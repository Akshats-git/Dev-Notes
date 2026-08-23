# Abstraction and Polymorphism, Abstract Class vs Interface

## Abstraction

Abstraction means showing only the essential features of something and hiding the internal complexity. A driver uses a steering wheel and pedals without needing to know exactly how the engine or brakes work internally. In code, abstraction means exposing a simple interface to the caller while hiding implementation details behind it.

Java achieves abstraction in two ways: abstract classes and interfaces.

## Abstract classes

An abstract class is a class that cannot be instantiated directly, it exists only to be extended. It can contain a mix of abstract methods (declared but with no body, implementation deferred to subclasses) and concrete methods (with a full body, shared as-is by all subclasses).

```java
abstract class Shape {
    String name;

    Shape(String name) { // abstract classes CAN have constructors
        this.name = name;
    }

    abstract double area(); // no body, subclasses must implement this

    void describe() { // concrete method, shared by all subclasses
        System.out.println(name + " has area " + area());
    }
}

class Circle extends Shape {
    double radius;
    Circle(double radius) {
        super("Circle");
        this.radius = radius;
    }
    double area() {
        return Math.PI * radius * radius;
    }
}
```

```java
Shape s = new Shape("test"); // compile error, cannot instantiate an abstract class
Shape c = new Circle(5);     // fine, Circle provides the missing implementation
c.describe();
```

### Why do abstract classes have constructors if they cannot be instantiated

This confuses people, but it makes sense once you remember that a subclass constructor always calls its parent's constructor first (see the inheritance note). The abstract class's constructor still runs, just as part of building a subclass object, to initialize the fields the abstract class owns. It is never called on its own through `new AbstractClass()`, only indirectly through `super(...)` from a subclass constructor.

## Interfaces

An interface defines a contract, a set of methods that implementing classes must provide, without dictating how they are implemented. Historically (before Java 8), an interface could only have abstract method signatures and constants.

```java
interface Playable {
    void play(); // implicitly public and abstract
}

class Song implements Playable {
    public void play() {
        System.out.println("Playing song");
    }
}
```

### Interfaces evolved a lot, and courses sometimes describe outdated rules

A statement like "interfaces can only have abstract methods" was true before Java 8, but it has not been the full picture for a long time now. This is worth correcting clearly since it comes up often.

- **Java 8** added **default methods**, which do have a body, giving interfaces the ability to provide shared behavior, and letting you add new methods to an interface without breaking existing implementing classes.
```java
interface Playable {
    void play();
    default void pause() {
        System.out.println("Paused");
    }
}
```
- **Java 8** also added **static methods** in interfaces, callable directly on the interface name, useful for utility logic related to the interface.
```java
interface MathOps {
    static int square(int x) { return x * x; }
}
int r = MathOps.square(4);
```
- **Java 9** added **private methods** in interfaces, used to share code between default methods inside the same interface, without exposing that helper method to implementing classes.

All fields declared in an interface are implicitly `public static final`, meaning they are constants, even if you do not write those keywords yourself.

## Abstract class vs interface

| | Abstract class | Interface |
|---|---|---|
| Instantiable | No | No |
| Fields | Any kind, including instance fields | Implicitly public static final (constants only) |
| Constructors | Yes | No |
| Method implementation | Yes, freely | Yes, since Java 8, via default/static/private methods |
| A class can extend/implement how many | Only one abstract class (single inheritance) | Any number of interfaces |
| Access modifiers on methods | Any | Implicitly public for abstract methods |
| Use case | Sharing a common base with some shared state and some shared logic among closely related classes | Defining a capability or contract that unrelated classes can all promise to fulfill |

### When to choose one over the other

Use an abstract class when subclasses share a strong "IS-A" relationship and some common state or implementation, for example `Dog` and `Cat` both extending an `Animal` abstract class that stores a shared `name` field and a shared `eat()` method. Use an interface when you want to define a capability that many unrelated classes can implement, for example `Comparable`, `Serializable`, or a custom `Flyable` interface implemented by both `Bird` and `Airplane`, which are otherwise completely unrelated classes.

## Polymorphism

Polymorphism means "many forms". The same method call can behave differently depending on the context. Java has two kinds.

### Compile time polymorphism: method overloading

Multiple methods in the same class share a name but differ in parameter list (different number or types of parameters). Java decides which one to call at compile time, based on the arguments you pass.

```java
class Printer {
    void print(String s) { System.out.println("String: " + s); }
    void print(int i) { System.out.println("Int: " + i); }
    void print(String s, int i) { System.out.println(s + " " + i); }
}
```

Return type alone is not enough to distinguish overloaded methods, two methods with the same name and same parameter list but different return types will not compile.

### Runtime polymorphism: method overriding

A subclass provides its own implementation of a method already defined in its superclass, with the exact same signature. Which version runs is decided at runtime, based on the actual object type, not the reference type. This is also called dynamic method dispatch.

```java
class Animal {
    void sound() { System.out.println("Some generic sound"); }
}
class Dog extends Animal {
    @Override
    void sound() { System.out.println("Bark"); }
}
class Cat extends Animal {
    @Override
    void sound() { System.out.println("Meow"); }
}

Animal a1 = new Dog();
Animal a2 = new Cat();
a1.sound(); // "Bark", decided at runtime by actual object type
a2.sound(); // "Meow"
```

Even though both `a1` and `a2` are declared as type `Animal`, the actual method that runs depends on what object they really point to. Compare this with static method hiding covered in the previous note, where the reference type decides, not the object type. This difference is a favorite interview question.

### Rules for valid overriding

- Method name, parameter list, and return type (or a covariant subtype of it) must match exactly.
- The access modifier in the subclass cannot be more restrictive than in the superclass, you can widen access (protected to public) but not narrow it (public to protected).
- A `static`, `final`, or `private` method cannot be overridden. Static methods are hidden, not overridden, as explained in the static and final note. Final methods explicitly forbid overriding. Private methods are not visible to subclasses at all, so a "same signature" method in a subclass is just a new unrelated method, not an override.
- Constructors are never overridden, since they are not inherited.

## Common mistakes to correct

- Saying "interfaces cannot have method bodies" is outdated. Since Java 8, default and static methods can have full implementations, and since Java 9, private methods too.
- Saying "overloading and overriding are basically the same idea" understates the difference. Overloading is resolved at compile time using the parameter list. Overriding is resolved at runtime using the actual object type. They solve different problems.
- Saying "you choose abstract class or interface based on whether you need methods with a body" is now outdated advice, since interfaces can have method bodies too. The real deciding factor today is about the type of relationship (IS-A with shared state versus a pluggable capability) and whether you need multiple inheritance of type.

## Interview questions

**Q: What is the real difference between method overloading and method overriding?**
Overloading is having multiple methods with the same name but different parameter lists in the same class, resolved at compile time. Overriding is a subclass redefining a method with the exact same signature as its superclass, resolved at runtime based on the actual object.

**Q: Can an abstract class have a constructor, and when does it run?**
Yes. It runs whenever a subclass object is created, as part of the constructor chain, even though the abstract class itself can never be instantiated directly.

**Q: Since interfaces can have default methods now, what is still different from an abstract class?**
Interfaces still cannot hold instance state (only constants), cannot have constructors, and a class can implement many interfaces but extend only one abstract class.

**Q: What happens if a class implements two interfaces that both have a default method with the same signature?**
The compiler forces the implementing class to explicitly override that method itself, resolving the conflict, rather than picking one automatically.

**Q: Can a private method be overridden?**
No. Since it is not visible outside its own class, a subclass defining a method with the same signature is just creating a separate, unrelated method, not overriding anything.

## Quick summary

- Abstraction hides implementation complexity, achieved through abstract classes and interfaces.
- Abstract classes can mix abstract and concrete methods, have constructors and instance fields, but support only single inheritance.
- Interfaces define contracts, support multiple implementation per class, and since Java 8 can include default, static, and (since Java 9) private methods with real bodies.
- Compile time polymorphism is overloading, resolved by parameter list at compile time.
- Runtime polymorphism is overriding, resolved by actual object type at runtime, this is dynamic method dispatch.
- static, final, and private methods cannot be truly overridden.
