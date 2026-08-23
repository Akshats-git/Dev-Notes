# AutoBoxing, POJOs, and Why Only One Public Class Per File

## Wrapper classes

Java has two kinds of types: primitives (`int`, `double`, `boolean`, `char`, and so on) and reference types (objects). Primitives are fast and lightweight but cannot be used where an object is required, for example inside collections like `ArrayList`, which only store objects. To bridge this gap, Java provides a wrapper class for every primitive.

| Primitive | Wrapper class |
|---|---|
| int | Integer |
| double | Double |
| boolean | Boolean |
| char | Character |
| long | Long |
| float | Float |
| byte | Byte |
| short | Short |

## Autoboxing and unboxing

**Autoboxing** is the automatic conversion of a primitive value into its wrapper object, done by the compiler. **Unboxing** is the reverse, converting a wrapper object back into its primitive value automatically.

```java
int a = 10;
Integer boxed = a;      // autoboxing, compiler inserts Integer.valueOf(a)
int unboxed = boxed;    // unboxing, compiler inserts boxed.intValue()

List<Integer> list = new ArrayList<>();
list.add(5); // autoboxing happens here too, 5 becomes Integer.valueOf(5)
int first = list.get(0); // unboxing happens here
```

Before Java 5, you had to do this conversion manually with `Integer.valueOf(a)` and `boxed.intValue()`. Autoboxing just means the compiler now writes that conversion code for you behind the scenes.

## The Integer cache, a very common interview trap

This is one of the most frequently asked "gotcha" questions in Java interviews, and it is directly related to autoboxing, so it belongs here.

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b); // true

Integer x = 200;
Integer y = 200;
System.out.println(x == y); // false
```

Why does this happen? When autoboxing occurs, Java calls `Integer.valueOf(int)` internally, not `new Integer(int)`. `Integer.valueOf` maintains an internal cache of `Integer` objects for values from -128 to 127. For values in that range, it returns the same cached object every time, so `==` (which compares references, not values, for objects) returns true because both variables point to the exact same cached object. For values outside that range, a new `Integer` object is created each time, so `==` returns false, even though the actual numeric values are equal.

This is a real correctness trap in production code too, not just a trivia question. The safe rule: **always use `.equals()` to compare wrapper object values, never `==`.**

```java
Integer x = 200;
Integer y = 200;
System.out.println(x.equals(y)); // true, correct way to compare values
```

`String` has a similar but separately implemented caching mechanism called the string pool, which follows its own rules, not the same as the Integer cache.

## NullPointerException risk from unboxing

Unboxing a `null` wrapper object throws a `NullPointerException`, because Java tries to call a method like `.intValue()` on `null`. This is an easy bug to introduce accidentally.

```java
Integer count = null;
int total = count; // throws NullPointerException at unboxing
```

This commonly happens with database or API results, where a field can legitimately be `null` (unknown or missing), but the code assigns it directly to a primitive expecting a value to always exist.

## POJO: Plain Old Java Object

A POJO is simply a Java class that does not extend any special framework class, does not implement any special framework interface, and does not carry any framework specific annotations forcing particular behavior. It typically has:

- Private fields.
- Public getters and setters for those fields.
- A no-argument constructor (often, though not strictly mandatory in every definition).
- No business logic tied to a specific framework (no database, no web, no dependency injection annotations forcing behavior).

```java
public class Employee {
    private String name;
    private double salary;

    public Employee() { }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public double getSalary() { return salary; }
    public void setSalary(double salary) { this.salary = salary; }
}
```

The term exists mainly as a contrast to heavier, framework-bound objects from older Java frameworks (like old-style EJBs), which forced you to extend specific base classes or implement specific lifecycle interfaces just to hold simple data. A POJO is deliberately plain and framework-independent, which makes it easy to test and reuse anywhere.

### POJO vs JavaBean, a distinction courses sometimes blur

These terms are often used interchangeably, but a JavaBean is technically a more specific and stricter kind of POJO. A JavaBean additionally requires:
- The class must implement `Serializable`.
- Fields must be accessed only through getters and setters following the exact `getX` / `setX` / `isX` (for booleans) naming convention.
- It must have a public no-argument constructor.

So every JavaBean is a POJO, but not every POJO is a JavaBean, since a POJO does not strictly require `Serializable` or the exact naming convention. If a course uses "POJO" and "JavaBean" as perfectly identical terms, that is a slight oversimplification worth knowing about.

## Why a Java file can only have one public class

If you try to put two public classes in the same `.java` file, the compiler rejects it with an error. A single file can contain multiple classes, but at most one of them can be `public`, and the file name must exactly match that public class's name.

```java
// File: Employee.java
public class Employee {
}

class Helper { // fine, non-public, can coexist in the same file
}

public class Another { } // compile error, only one public class allowed per file
```

The reason relates to how the JVM finds and loads classes. Java expects to find a public class in a file with the exact same name, `Employee.class` compiled from `Employee.java` should represent the public class `Employee`. This gives a clean, predictable one-to-one mapping between a public class name and the file (and later, the compiled `.class` file) that contains it, which the class loader relies on when locating classes by name. If two public classes could live in one file, the compiler and the class loader would have an ambiguous rule for which name maps to which file.

A common oversimplification is saying "the class name and file name must always match". That is only strictly enforced for the public class. Any number of non-public (package-private) classes can share the file with different names of their own, as shown by `Helper` above.

## Common mistakes to correct

- Saying "Integer values are always cached" is wrong, only the range -128 to 127 is cached by default. Outside that range, `==` comparison on wrapper objects is unreliable.
- Saying "autoboxing has no downside" ignores both the NullPointerException risk on unboxing null, and a real performance cost from the extra object creation happening invisibly in loops that repeatedly box and unbox large numbers of values.
- Saying "POJO and JavaBean mean exactly the same thing" is not precise. A JavaBean is a POJO with extra formal requirements (Serializable, strict naming, no-arg constructor).

## Interview questions

**Q: Why does Integer a = 127; Integer b = 127; a == b give true, but the same code with 200 gives false?**
Because autoboxing uses Integer.valueOf(), which caches Integer objects for values -128 to 127 and reuses the same cached object. Outside that range, a new object is created each time, so reference comparison with == returns false even though the values are equal.

**Q: What is the safe way to compare two Integer objects for equal value?**
Use .equals(), never ==, since == compares object references, not the numeric value, once autoboxing is involved.

**Q: What happens if you unbox a null Integer into an int?**
It throws a NullPointerException, since unboxing calls a method like intValue() on the wrapper object, and you cannot call a method on null.

**Q: What is a POJO?**
A plain Java class with no forced framework dependencies, typically with private fields and public getters and setters, usable independently of any specific framework.

**Q: Why can a Java file have only one public class?**
Because the JVM class loader relies on a direct mapping between a public class's name and its source and compiled file name, so allowing multiple public classes in one file would break that predictable mapping.

## Quick summary

- Autoboxing/unboxing automatically converts between primitives and their wrapper classes.
- Integer caching only applies to values from -128 to 127, always use .equals() to compare wrapper values.
- Unboxing a null wrapper throws NullPointerException.
- A POJO is a simple, framework-independent Java class, typically with private fields and public getters/setters.
- A JavaBean is a stricter POJO, requiring Serializable, strict naming conventions, and a no-arg constructor.
- A .java file can have only one public class, and its name must match the file name exactly.
