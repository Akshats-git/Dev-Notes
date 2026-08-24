# Encapsulation and Inheritance, Packages and the super Keyword

## Encapsulation

Encapsulation means bundling data (fields) and the code that operates on that data (methods) together inside one unit, a class, and restricting direct outside access to the internal data. This is often called data hiding.

The standard pattern: make fields `private`, and provide `public` getter and setter methods to read and update them in a controlled way.

```java
class BankAccount {
    private double balance; // hidden from outside

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    public void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
        }
    }
}
```

Without encapsulation, if `balance` were public, any outside code could set it to a negative number directly, `account.balance = -500;`, bypassing all the validation logic. With `balance` private, the only way to change it is through `deposit` and `withdraw`, which enforce the rules. This is the real practical value of encapsulation, it is not just about hiding fields for its own sake, it is about controlling how data can be changed.

### Access modifiers

Java has four access levels, from most to least restrictive.

| Modifier | Same class | Same package | Subclass (different package) | Everywhere |
|---|---|---|---|---|
| private | yes | no | no | no |
| default (no keyword) | yes | yes | no | no |
| protected | yes | yes | yes | no |
| public | yes | yes | yes | yes |

A detail often left vague: `protected` gives access to subclasses even outside the package, but only through inheritance, a subclass in another package can access a protected member of its own inherited instance, but not through an unrelated object of the parent type.

## Packages

A package is a way to group related classes together, similar to folders on a file system. Packages help avoid naming conflicts and organize large codebases.

```java
package com.example.bank;

import java.util.List; // importing a class from another package
```

Classes in the same package can access each other's package-private (default) members without any import. Classes in different packages need an `import` statement, or must use the fully qualified name.

## Inheritance

Inheritance lets one class acquire the fields and methods of another class. The class being inherited from is the superclass (or parent, or base class). The class doing the inheriting is the subclass (or child, or derived class). This models an "IS-A" relationship, for example a `Car` IS-A `Vehicle`.

```java
class Vehicle {
    int speed;
    void move() {
        System.out.println("Vehicle is moving");
    }
}

class Car extends Vehicle {
    String model;
}

Car myCar = new Car();
myCar.speed = 100; // inherited field
myCar.move();      // inherited method
```

Why use inheritance: it lets you reuse code instead of rewriting it, and it lets you write general code that works with any subtype (this second benefit is really polymorphism, covered in the next note, and inheritance is what makes it possible).

### Types of inheritance

- **Single inheritance**: one subclass, one superclass. `Car extends Vehicle`.
- **Multilevel inheritance**: a chain, `SportsCar extends Car extends Vehicle`.
- **Hierarchical inheritance**: multiple subclasses share one superclass, `Car extends Vehicle` and `Bike extends Vehicle`.

### Java does not support multiple inheritance of classes, and why

A class cannot extend two classes at once, `class X extends A, B` is not legal Java. The reason is a problem called the diamond problem: if class `A` and class `B` both define a method `doWork()` with different implementations, and class `X` extends both, the compiler would not know which version `X` should inherit. Java avoids this ambiguity entirely by disallowing multiple class inheritance.

### Important correction: Java does support a form of multiple inheritance

A course sometimes states flatly "Java does not support multiple inheritance", without qualification. This is incomplete and slightly misleading. The precise statement is: **Java does not support multiple inheritance through classes, but it does support multiple inheritance of type through interfaces.** A class can implement any number of interfaces at once.

```java
interface Flyable { void fly(); }
interface Swimmable { void swim(); }

class Duck implements Flyable, Swimmable {
    public void fly() { System.out.println("Duck flying"); }
    public void swim() { System.out.println("Duck swimming"); }
}
```

Java 8 introduced default methods in interfaces, which can carry actual implementation, so the diamond problem can technically appear with interfaces too, when two interfaces provide the same default method. Java handles this by forcing the implementing class to override the conflicting method itself, rather than guessing which one to use. This is covered in more depth in the interfaces note.

## The super keyword

`super` refers to the immediate parent class. It has two main uses.

### 1. Accessing a parent's field or method that is hidden or overridden

```java
class Vehicle {
    int speed = 50;
    void move() { System.out.println("Vehicle moving"); }
}

class Car extends Vehicle {
    int speed = 100; // hides the parent's speed field

    void show() {
        System.out.println(speed);       // 100, this class's own field
        System.out.println(super.speed); // 50, parent's field
        super.move();                    // calls parent's version of move()
    }
}
```

### 2. Calling the parent's constructor

```java
class Vehicle {
    int speed;
    Vehicle(int speed) {
        this.speed = speed;
        System.out.println("Vehicle constructed");
    }
}

class Car extends Vehicle {
    String model;
    Car(String model, int speed) {
        super(speed); // must be the first statement
        this.model = model;
        System.out.println("Car constructed");
    }
}
```

### Constructor chain in inheritance, an important detail

Every subclass constructor implicitly calls the parent's no-argument constructor as its very first action, unless you explicitly write a different `super(...)` call yourself. This means the parent part of an object is always fully built before the child's own constructor logic runs. This holds true no matter how many levels of inheritance exist, the chain always runs from the top of the hierarchy downward.

```java
class A { A() { System.out.println("A"); } }
class B extends A { B() { System.out.println("B"); } }
class C extends B { C() { System.out.println("C"); } }

new C();
// Output: A, then B, then C
```

If the parent class has no no-argument constructor available (only a parameterized one), and the subclass does not explicitly call `super(someArgument)`, the code will not compile. This is a very common beginner compile error.

## Are private members inherited

This is a subtle point worth being precise about. Technically, a subclass object does contain the private fields of its parent in memory, they are still part of the object. But the subclass code cannot access them directly by name, because `private` restricts access to the declaring class only. The subclass can only interact with them indirectly, through public or protected getter/setter methods the parent provides. So the common short answer "private members are not inherited" is a simplification, more accurately: they exist in the object but are not accessible to the subclass directly.

## Common mistakes to correct

- Saying "Java does not support multiple inheritance" without the interface qualification is incomplete. Multiple inheritance of type through interfaces is fully supported.
- Saying "private fields are not inherited" is a simplification. They are present in the subclass object but not directly accessible by name.
- Forgetting that the parent constructor always runs before the child constructor body, even when not written explicitly.

## Interview questions

**Q: Why does Java not allow a class to extend two classes?**
To avoid the diamond problem, where the compiler cannot decide which parent's conflicting method implementation to inherit. Java sidesteps this by allowing only single inheritance of classes.

**Q: How does Java achieve multiple inheritance then?**
Through interfaces. A class can implement multiple interfaces, achieving multiple inheritance of type, and default methods make this cleaner since Java 8.

**Q: What is the order of constructor execution in an inheritance chain?**
The topmost superclass constructor runs first, then each level down to the subclass, ending with the subclass's own constructor body.

**Q: Can a subclass access a private field of its superclass?**
Not directly by name. It can only access it indirectly through public or protected methods the superclass exposes, like getters and setters.

**Q: What is the difference between encapsulation and abstraction?**
Encapsulation is about hiding internal data and controlling access to it through methods. Abstraction is about hiding implementation complexity and exposing only essential behavior. They are related but distinct, abstraction is covered fully in the next note.

## Quick summary

- Encapsulation hides internal data behind private fields and public getters/setters, giving controlled access.
- Access modifiers, from most to least restrictive: private, default, protected, public.
- Inheritance lets a subclass reuse a superclass's fields and methods, modeling an IS-A relationship.
- Java disallows multiple inheritance through classes to avoid the diamond problem, but supports it through interfaces.
- super accesses a parent's hidden field, overridden method, or constructor. Parent constructors always run before child constructors.
