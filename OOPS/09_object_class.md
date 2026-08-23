# The Object Class: equals, hashCode, toString

## Object is the root of every class

Every class in Java, whether you write `extends` or not, implicitly extends `java.lang.Object`. This means every single object you ever create in Java automatically has a base set of methods available, inherited from `Object`.

```java
class Car { } // secretly, this is "class Car extends Object { }"
```

The most commonly discussed methods from `Object` are `equals()`, `hashCode()`, and `toString()`. There are others too, like `getClass()`, `clone()`, `wait()`, `notify()`, `notifyAll()`, and the deprecated `finalize()`, but the three above are by far the most relevant for everyday code and interviews.

## toString()

The default `toString()` implementation returns a string in the form `ClassName@hexadecimalHashCode`, which is not very useful for debugging or logging.

```java
class Car {
    String model;
    Car(String model) { this.model = model; }
}

Car c = new Car("Tesla");
System.out.println(c); // something like Car@1b6d3586, not helpful
```

Overriding `toString()` gives a meaningful, readable representation, and it is called automatically whenever an object is printed or concatenated with a string.

```java
class Car {
    String model;
    Car(String model) { this.model = model; }

    @Override
    public String toString() {
        return "Car{model='" + model + "'}";
    }
}

Car c = new Car("Tesla");
System.out.println(c); // Car{model='Tesla'}
```

## equals()

The default `equals()` implementation, inherited straight from `Object`, simply checks reference equality, it behaves exactly like `==`, checking whether both variables point to the exact same object in memory.

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
}

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
System.out.println(p1.equals(p2)); // false by default, different objects even though same data
```

Most of the time you actually want "equal data" to mean "equal", not "exact same object". To get that, you override `equals()`.

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Point other = (Point) obj;
        return this.x == other.x && this.y == other.y;
    }
}
```

### Rules a correct equals() implementation must follow

- **Reflexive**: `x.equals(x)` must be true.
- **Symmetric**: if `x.equals(y)` is true, then `y.equals(x)` must also be true.
- **Transitive**: if `x.equals(y)` and `y.equals(z)` are true, then `x.equals(z)` must be true.
- **Consistent**: repeated calls should give the same result, as long as the fields used in the comparison do not change.
- **Null comparison**: `x.equals(null)` must return false, never throw an exception.

## hashCode()

`hashCode()` returns an integer meant to represent the object for use in hash-based collections like `HashMap`, `HashSet`, and `HashTable`. The default implementation, inherited from `Object`, is typically based on the object's memory address, so two different objects almost always get different hash codes by default, even with identical data.

## The equals and hashCode contract, the single most important point in this topic

This is a rule that is easy to state but genuinely important to understand deeply, because getting it wrong causes real, hard to diagnose bugs. **If two objects are equal according to equals(), they MUST have the same hashCode(). The reverse is not required, two unequal objects are allowed to share the same hashCode (called a hash collision), though it is best if this is rare.**

If a course explains `equals()` in isolation without stressing this contract with `hashCode()`, that is a significant gap, since overriding one without the other is a very common real bug.

### What breaks if you override equals() but not hashCode()

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Point other = (Point) obj;
        return this.x == other.x && this.y == other.y;
    }
    // hashCode() NOT overridden, still uses Object's default based on memory address
}

Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));
System.out.println(set.contains(new Point(1, 2))); // false! even though equals() says they are equal
```

This looks broken and confusing at first, but it makes sense once you understand how `HashSet` and `HashMap` actually work internally. They use `hashCode()` first to decide which internal "bucket" to look in, and only then use `equals()` to compare objects within that bucket. Since the two `Point` objects here have different default hash codes (based on memory address), the set looks in the wrong bucket entirely and never even calls `equals()` to compare them. The fix is to override `hashCode()` consistently with `equals()`, using the same fields.

```java
@Override
public int hashCode() {
    return Objects.hash(x, y); // combines both fields into one hash code
}
```

With this fix, `set.contains(new Point(1, 2))` correctly returns `true`, since both `Point` objects now produce the same hash code and land in the same bucket, allowing `equals()` to correctly find the match.

## Objects utility class, extra useful info

`java.util.Objects` provides convenient null-safe helper methods that are commonly used inside `equals()` and `hashCode()` implementations, and most IDEs generate code using them automatically.

```java
import java.util.Objects;

@Override
public boolean equals(Object obj) {
    if (this == obj) return true;
    if (!(obj instanceof Point)) return false;
    Point other = (Point) obj;
    return x == other.x && y == other.y;
}

@Override
public int hashCode() {
    return Objects.hash(x, y);
}
```

`Objects.equals(a, b)` is also useful for safely comparing two references that might be null, without risking a `NullPointerException`, unlike calling `a.equals(b)` directly when `a` could be null.

## Common mistakes to correct

- Explaining equals() without clearly stating the mandatory link to hashCode() leaves out the single most practically important rule in this topic, and is the direct cause of the classic "HashSet contains returns false even though equals says true" bug shown above.
- Saying "== and equals() always do different things" is not quite right either, by default (before any override) equals() behaves exactly like ==. They only diverge once equals() is overridden.
- Saying "you should always override equals() for every class" is too broad. Immutable value-like classes (representing pure data, like a Point or a Money amount) benefit from it. Classes representing unique, ongoing entities where identity truly means memory identity (many service or manager style classes) often should keep the default reference-based equals().

## Interview questions

**Q: What is the equals and hashCode contract?**
If two objects are equal according to equals(), they must return the same hashCode(). The reverse is not required, unequal objects may share a hash code.

**Q: What goes wrong if you override equals() but forget hashCode()?**
Hash based collections like HashMap and HashSet will behave incorrectly, since they use hashCode() to locate the right bucket before calling equals(). Two "equal" objects with different default hash codes will land in different buckets and never be recognized as equal by the collection, even though calling .equals() on them directly returns true.

**Q: What does equals() do by default if you never override it?**
It behaves exactly like ==, comparing object references, true only if both variables point to the exact same object in memory.

**Q: Name the five properties a correct equals() implementation must satisfy.**
Reflexive, symmetric, transitive, consistent, and it must return false (not throw) when compared to null.

**Q: What does toString() return by default?**
ClassName@hexadecimalHashCode, which is why it is almost always overridden for meaningful debugging or logging output.

## Quick summary

- Every class implicitly extends Object and inherits equals(), hashCode(), toString(), and more.
- Default equals() is reference equality, same as ==. Default hashCode() is typically based on memory address. Default toString() prints ClassName@hashcode.
- If you override equals(), you must also override hashCode() consistently, or hash based collections will behave incorrectly.
- Objects.equals() and Objects.hash() are convenient null-safe helpers commonly used inside these overrides.
