# Constructors, Constructor Chaining, Overloading and this

## What is a constructor

A constructor is a special block of code that runs automatically when an object is created with `new`. Its job is to initialize the object's fields.

Rules for a constructor:
- Its name must exactly match the class name.
- It has no return type, not even `void`.
- It runs only once per object, at creation time.

```java
class Car {
    String model;
    int speed;

    Car(String model, int speed) {
        this.model = model;
        this.speed = speed;
    }
}

Car myCar = new Car("Tesla", 120);
```

## Constructor is not a method

This point is worth stating clearly because it is sometimes glossed over. A constructor looks similar to a method but it is a different thing.

| Constructor | Method |
|---|---|
| Same name as the class | Any name you choose |
| No return type at all | Must have a return type, or void |
| Called automatically by `new` | Called explicitly by you |
| Runs once per object | Can run many times |

A common trap: if you write `void Car() { }` inside class `Car`, this is NOT a constructor. It is a regular method that happens to share the class name, because it has a return type (`void`). Java will not call it automatically when you do `new Car()`.

## Default constructor

If you do not write any constructor, the Java compiler automatically inserts a no-argument constructor for you that does nothing but call the parent constructor. This is called the default constructor.

```java
class Car {
    String model;
    // no constructor written here
}
// compiler behaves as if you wrote:
// Car() { super(); }
```

Important: the moment you write even one constructor yourself, Java stops adding the default one. So if you write only a parameterized constructor, `new Car()` with no arguments will no longer compile unless you also write a no-argument constructor yourself.

```java
class Car {
    String model;
    Car(String model) {
        this.model = model;
    }
}
Car c = new Car(); // compile error, no matching constructor
```

### What "the parent constructor" means

Every constructor in Java must, as its very first action, run a constructor of its superclass. If you do not write that call yourself, the compiler inserts `super();` for you. So the "parent constructor" is simply the constructor of the class you extend.

```java
class Dog {
    Dog() { }          // what you wrote
}

class Dog {
    Dog() {
        super();       // what the compiler actually compiles
    }
}
```

### But what if the class has no parent

It always has one. If a class does not say `extends`, the compiler treats it as `extends Object`. `Object` is the root of every class hierarchy and it has a public no-argument constructor, so the implicit `super()` resolves to `Object()`, which does essentially nothing. The chain always terminates at `Object`:

```
Puppy() -> Dog() -> Animal() -> Object()
```

So the question "what if there is no parent constructor" never applies to the *absence of a parent*. It applies to a parent that has no **no-argument** constructor, which is a different and much more common situation.

### The case that actually breaks compilation

Remember the rule from above: the compiler adds a default constructor only if you wrote no constructor at all. So the moment a parent declares a parameterized constructor, it loses its free no-arg one.

```java
class Animal {
    String name;
    Animal(String name) {     // a constructor is written...
        this.name = name;     // ...so Animal() no longer exists
    }
}

class Dog extends Animal {    // compiler tries to insert Dog() { super(); }
}
```

This does not compile:

```
error: constructor Animal in class Animal cannot be applied to given types;
  required: String
  found:    no arguments
```

The implicit `super()` has nothing to bind to. The same error appears if the parent's no-arg constructor exists but is `private`, it exists but is not accessible from the subclass.

Two ways to fix it. Either call the parent constructor explicitly:

```java
class Dog extends Animal {
    Dog(String name) {
        super(name);          // explicit, so no implicit super() is inserted
    }
}
```

Or give the parent a no-argument constructor of its own:

```java
class Animal {
    String name;
    Animal() { this.name = "unknown"; }   // now the implicit super() works again
    Animal(String name) { this.name = name; }
}
```

### How this() interacts with it

`this(...)` also counts as the first statement, so it suppresses the implicit `super()` and defers the parent call to whichever constructor it delegates to.

```java
class Dog extends Animal {
    Dog() {
        this("Rex");          // no implicit super() here
    }
    Dog(String name) {
        super(name);          // the parent constructor runs here instead
    }
}
```

The parent constructor still runs exactly once, just one step later in the chain. Full detail on the inheritance chain and its execution order is in the inheritance note.

### A small accuracy point on the default constructor

The default constructor is often described as "public". More precisely, it gets the same access level as the class itself. A `public class` gets a public default constructor, a package-private class gets a package-private one.

## Constructor overloading

A class can have multiple constructors as long as their parameter lists are different (different number or different types of parameters). This is called constructor overloading.

```java
class Car {
    String model;
    int speed;

    Car() {
        model = "Unknown";
        speed = 0;
    }

    Car(String model) {
        this.model = model;
        speed = 0;
    }

    Car(String model, int speed) {
        this.model = model;
        this.speed = speed;
    }
}
```

Java decides which constructor to call by matching the number and types of arguments you pass, at compile time. This is a form of compile time polymorphism, same idea as method overloading.

## Constructor chaining with this()

Instead of repeating initialization code in every constructor, one constructor can call another constructor of the same class using `this(...)`.

```java
class Car {
    String model;
    int speed;

    Car() {
        this("Unknown", 0); // calls the two-arg constructor
    }

    Car(String model) {
        this(model, 0); // calls the two-arg constructor
    }

    Car(String model, int speed) {
        this.model = model;
        this.speed = speed;
    }
}
```

Strict rule: `this(...)` must be the very first statement in a constructor. You cannot write any code before it. You also cannot use both `this(...)` and `super(...)` in the same constructor, since both must be the first statement and only one first statement is allowed.

## The this keyword, full picture

`this` refers to the current object, the one whose method or constructor is currently running. It has three common uses:

1. To distinguish a field from a parameter with the same name.
```java
Car(String model) {
    this.model = model;
}
```
2. To call another constructor of the same class (constructor chaining), shown above.
3. To pass the current object as an argument to another method.
```java
void register(CarRegistry registry) {
    registry.add(this);
}
```

## super() and constructors, a quick preview

If a class extends another class, every constructor implicitly calls the parent class's no-argument constructor first, unless you explicitly call `super(...)` or `this(...)` yourself. This means the parent object is always fully constructed before the child's own constructor body runs. Full detail with examples is in the inheritance note.

## Common mistake to correct

A mistake sometimes made is saying "if you don't write a constructor, your object won't work". This is wrong. Java always guarantees a default constructor exists as long as you have not written any constructor yourself. It is also sometimes claimed that a class "must have a constructor with the same name written by the programmer". That is not true either, the compiler handles it silently when no constructor is written.

Another mistake is thinking constructor overloading and constructor chaining are the same thing. They are not. Overloading is about having multiple constructors with different signatures. Chaining is about one constructor calling another using `this(...)` to avoid duplicate code. You can have overloading without chaining, and in small classes you often should, chaining is just a way to reduce repetition.

## Private constructors, extra useful info

A constructor can be marked `private`. This means the class cannot be instantiated with `new` from outside the class. This is commonly used in the Singleton design pattern, where a class controls creation of its own single instance.

```java
class Singleton {
    private static Singleton instance;
    private Singleton() { }

    static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

This is a favorite interview follow up question after constructors, so it is worth remembering.

## Interview questions

**Q: Can a constructor be private? What is it used for?**
Yes. It is commonly used to prevent object creation from outside the class, as seen in the Singleton pattern.

**Q: What happens if you do not write any constructor?**
The compiler inserts a no-argument default constructor automatically, whose body is just a call to the parent constructor. Its access level matches the class's own.

**Q: What is the parent constructor that the default constructor calls?**
The constructor of the superclass. If the class does not extend anything, its superclass is `Object`, so `super()` calls `Object()`. Every constructor chain ends at `Object`.

**Q: What happens if the parent class has no no-argument constructor?**
The implicit `super()` has nothing to bind to and the code fails to compile. You must either call `super(args)` explicitly in the subclass constructor, or add a no-argument constructor to the parent.

**Q: If you write a parameterized constructor, does the default no-arg constructor still exist?**
No. Once you write any constructor yourself, Java stops generating the default one. You must write your own no-arg constructor if you still need one.

**Q: Can a constructor be inherited?**
No. Constructors are not inherited by subclasses. But a subclass constructor always calls a parent constructor first, using an implicit or explicit `super()`.

**Q: Can this() and super() both appear in the same constructor?**
No. Only one of them can be used, and it must be the first line of the constructor.

**Q: Is a constructor a special kind of method?**
No, treat it as a different construct. It has no return type and is invoked only through object creation, not by a normal method call.

## Quick summary

- Constructor initializes an object, name matches class, no return type.
- If no constructor is written, Java provides a no-arg default constructor, with the same access level as the class.
- Writing any constructor removes the automatic default one.
- Constructor overloading means multiple constructors with different parameter lists.
- Constructor chaining means one constructor calling another using this(...), must be the first line.
- Every constructor starts with an implicit super(), so the parent is always built first. A class with no extends has Object as its parent, and every chain ends at Object().
- If the parent has no accessible no-arg constructor, that implicit super() is a compile error. Fix it with an explicit super(args) or by adding a no-arg constructor to the parent.
- this refers to the current object, used to resolve naming conflicts, chain constructors, or pass the object itself somewhere.
