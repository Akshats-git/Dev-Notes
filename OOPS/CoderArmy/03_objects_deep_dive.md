# Objects Deep Dive: Size, Call by Value vs Reference, Shallow vs Deep Copy

## Java is always pass by value, correcting a very common myth

Many courses and articles say "primitives are passed by value, objects are passed by reference" in Java. This is a common but incorrect way to describe it, and it is worth correcting clearly because interviewers love this question.

The accurate statement is: **Java is always pass by value. There is no pass by reference in Java.**

The confusion happens because for objects, the "value" that gets copied and passed is the reference (the address) itself, not the object. So it looks like the object is being passed directly, but actually a copy of the reference is passed.

```java
class Box {
    int size;
}

void changeSize(Box b) {
    b.size = 100; // this changes the real object, both variables point to same object
}

void reassign(Box b) {
    b = new Box(); // this only changes the local copy of the reference, caller is unaffected
    b.size = 999;
}

public static void main(String[] args) {
    Box myBox = new Box();
    myBox.size = 5;

    changeSize(myBox);
    System.out.println(myBox.size); // prints 100, object was mutated through the shared reference

    reassign(myBox);
    System.out.println(myBox.size); // still prints 100, reassign only changed its own local copy of the reference
}
```

This example proves Java passes a copy of the reference, not the original reference itself. If it were true pass by reference, `reassign` would have changed what `myBox` in `main` points to as well, but it does not.

## Why this matters

Understanding this precisely lets you correctly predict two different behaviors:
- Mutating fields through a passed reference does affect the original object.
- Reassigning the parameter variable itself to a new object does not affect the caller's variable.

## Object size, a realistic view

Objects do not have a fixed universal size in Java. Their size depends on the JVM implementation, the fields they contain, and memory alignment. As a rough mental model on a typical 64-bit JVM:

- Every object has an object header, usually around 12 to 16 bytes, used internally by the JVM for things like the class pointer and lock/GC information.
- Each field adds its own size (int is 4 bytes, long is 8 bytes, a reference is typically 4 bytes with compressed pointers or 8 bytes without).
- The JVM often rounds the total size up to a multiple of 8 bytes, called padding.

You do not need to memorize exact numbers for an interview. What matters is understanding that object size is not something you calculate by hand casually, it depends on JVM internals, and tools like `Instrumentation.getObjectSize()` or profilers are used to measure it precisely when needed.

## Shallow copy vs deep copy

When you "copy" an object that itself contains other objects as fields, you must decide how deep the copy should go.

**Shallow copy**: creates a new object, but any reference type fields inside it still point to the same nested objects as the original. The top level object is duplicated, but nested objects are shared.

**Deep copy**: creates a new object and also recursively creates new copies of every nested object, so nothing is shared between the original and the copy.

```java
class Engine {
    int horsepower;
    Engine(int hp) { this.horsepower = hp; }
}

class Car {
    String model;
    Engine engine;

    Car(String model, Engine engine) {
        this.model = model;
        this.engine = engine;
    }

    // shallow copy constructor
    Car(Car other) {
        this.model = other.model;
        this.engine = other.engine; // same Engine object is shared, not copied
    }

    // deep copy constructor
    Car deepCopy() {
        Engine newEngine = new Engine(this.engine.horsepower);
        return new Car(this.model, newEngine);
    }
}
```

```java
Engine e = new Engine(300);
Car original = new Car("Tesla", e);

Car shallow = new Car(original); // shallow copy
shallow.engine.horsepower = 999;
System.out.println(original.engine.horsepower); // prints 999, both share the same Engine object, a real bug source

Car deep = original.deepCopy(); // deep copy
deep.engine.horsepower = 111;
System.out.println(original.engine.horsepower); // still 999, unaffected because deep copy has its own Engine
```

This example shows the classic bug caused by shallow copies: modifying the copy accidentally modifies the original, because they still share nested objects. This is one of the most practically useful things to understand deeply, since it causes real bugs in production code, not just interview trivia.

Note that `model`, a String, behaves safely either way because Strings are immutable in Java. Once created, a String's content can never change, so sharing a String reference between two objects is never a source of bugs. The danger is specifically with mutable nested objects like `Engine` here.

## Ways to copy objects in Java

1. **Manual copy constructor** (shown above), full control, most readable, recommended approach in most real code.
2. **clone() method and the Cloneable interface**. Java has a built in `clone()` method on `Object`, but it is widely considered poorly designed and many experienced developers avoid it. By default `Object.clone()` performs a shallow copy. To use it your class must implement the `Cloneable` marker interface, otherwise it throws `CloneNotSupportedException`. For a true deep copy you must override `clone()` yourself and manually clone every mutable field.
3. **Copy libraries or serialization based deep copy**, used sometimes for complex object graphs, but slower and adds overhead.

```java
class Engine implements Cloneable {
    int horsepower;
    Engine(int hp) { this.horsepower = hp; }

    @Override
    protected Engine clone() {
        try {
            return (Engine) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## Common mistake to correct

Saying "Java objects are passed by reference" is the single most common mistake in this topic and you should be ready to correct it confidently, using the example above as proof. The precise statement is: Java passes references by value.

Another common oversimplification is saying "clone() gives you a deep copy". By default it does not, `Object.clone()` is shallow. Whether you get a deep copy depends entirely on whether you override it to clone nested mutable fields yourself.

## Interview questions

**Q: Is Java pass by value or pass by reference?**
Java is strictly pass by value. For object type parameters, the value being copied is the reference itself, not the object. This is why mutating the object through the parameter affects the caller, but reassigning the parameter to a new object does not.

**Q: What is the difference between shallow copy and deep copy?**
A shallow copy duplicates the top level object but shares nested mutable objects with the original. A deep copy duplicates the top level object and recursively duplicates every nested object too, so nothing is shared.

**Q: Does Object.clone() give a deep copy by default?**
No, it gives a shallow copy by default. To get a true deep copy you must override clone() and manually deep copy the mutable fields.

**Q: Why are Strings safe to share between shallow copies?**
Because String is immutable in Java. Its content cannot change after creation, so two objects sharing the same String reference cannot cause the kind of bug that sharing a mutable object causes.

## Quick summary

- Java is always pass by value, for objects the value passed is a copy of the reference.
- Mutating an object's fields through a reference parameter affects the caller's object.
- Reassigning a reference parameter to a new object does not affect the caller's variable.
- Object size depends on header, field sizes, and JVM padding, not a fixed constant.
- Shallow copy shares nested mutable objects, deep copy duplicates them fully.
- Object.clone() is shallow by default and needs a custom Cloneable implementation for deep copies.
