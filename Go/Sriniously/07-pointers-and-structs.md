# 07 — Pointers & Structs

---

## Reference types

[▶ 2:03:27](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=7407s)

Go has **five reference types**.

| Type | Covered in |
|---|---|
| **Pointer** | this file |
| **Slice** | [08 — Arrays & slices](08-arrays-and-slices.md) |
| **Map** | [09 — Maps](09-maps.md) |
| **Channel** | [14 — Concurrency](14-concurrency.md) |
| **Function** | [10 — Functions as values](10-functions-as-values.md) |

They are called reference types because **they do not hold the data themselves**. They refer to data working in the background.

All five have a zero value of `nil`.

---

## Pointers

A pointer stores the **memory address** of a variable.

Here is the mental model. Your program's memory is a grid of slots. Every variable lives in a slot. Every slot has an address.

In Go it is a *virtual* address. The Go runtime keeps you away from the kernel. But you can reason about it as a physical address.

A pointer is a variable whose value *is* one of those addresses.

### The two operators

| Operator | Name | What it does |
|---|---|---|
| `&x` | address-of | gives you the address of `x` |
| `*p` | dereference | gives you the **value** stored at the address in `p` |

### Worked example

```go
i, j := 42, 2701

p := &i           // p is of type *int — "pointer to int"
fmt.Println(*p)   // 42 — read i through the pointer
*p = 21           // write to i through the pointer
fmt.Println(i)    // 21  ← i itself changed

p = &j            // point p at j instead
*p = *p / 37
fmt.Println(j)    // 73
```

Output: `42`, `21`, `73`.

### The key points

`p := &i` makes `p` a `*int`. **`p` is not an int.** It is a pointer that points to an int.

**Printing `p` directly prints the address.** You get a garbage-looking hex value, not the data. Print `*p` to see the value.

**Writing through a pointer changes the original.** `*p = 21` changed `i`.

Compare this to plain assignment. `p := i` copies the value. If you change `p` after that, `i` stays the same. With a pointer, both names refer to the same storage.

Pointers show up constantly. You will see them in method receivers, in avoiding large copies, and whenever a function needs to modify its caller's data. See [11 — Methods](11-methods.md).

---

## Structs

[▶ 2:10:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=7800s)

A struct is an **aggregate type**. It is a collection of **heterogeneous** fields, meaning fields of different types. An array is the other aggregate type, and it is homogeneous.

You will use structs constantly. Day-to-day Go backend work is mostly structs.

### Declaring one

```go
type Vertex struct {
	X int
	Y int
}
```

The pattern is `type <Name> struct { ... }`. Then field name, then field type.

If fields sit next to each other and share a type, you can merge them:

```go
type Vertex struct {
	X, Y int
}
```

### Four ways to initialize

```go
v1 := Vertex{1, 2}        // positional — X=1, Y=2, order matters
v2 := Vertex{X: 1}        // named fields — Y gets its zero value, 0
v3 := Vertex{}            // everything at zero value — X=0, Y=0
p  := &Vertex{1, 2}       // pointer to a struct — type is *Vertex
```

These print as `{1 2}`, `&{1 2}`, `{1 0}`, and `{0 0}`.

The named-field form is usually the better choice. It survives field reordering. And you do not have to remember positions.

### Reading and changing fields

```go
v := Vertex{1, 2}
fmt.Println(v.X)   // 1
v.X = 4
fmt.Println(v.X)   // 4
```

### Structs and pointers together

You will see this combination everywhere in real code.

```go
v := Vertex{1, 2}
p := &v
p.X = 1e9            // mutates v itself
fmt.Println(v)       // {1000000000 2}
```

Notice you write `p.X`, not `(*p).X`. **Go dereferences pointers to structs for you** when you access a field. It keeps the syntax clean. The same thing happens with method calls. See [11 — Methods](11-methods.md).

---

## Struct tags

[▶ ~2:16:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=8160s)

Real-world structs have backtick-quoted metadata after the field type.

```go
type BillingSubscription struct {
	PlanID string `json:"planID"`
	Status int    `json:"status"`
}
```

### Why they exist

Unlike JavaScript, **Go cannot use JSON directly.** JSON arrives as bytes. You have to *unmarshal* it into a Go struct before you can work with it.

But the naming conventions clash. JSON payloads use `planID` or `plan_id`. Go fields have to be `PascalCase` so they are exported and the JSON package can see them.

Struct tags are the **mapping layer** between the two.

```
{"planID": 3}   ──unmarshal──►   BillingSubscription{PlanID: 3}
                  guided by the `json:"planID"` tag
```

Different tag keys serve different libraries. You will see `json:`, `db:`, `validate:`, and `yaml:`. One struct can describe how it maps into several systems at once. fider's structs carry several tags depending on which package uses them.

---

## Exported struct fields

Field capitalization follows the same visibility rule as everything else. See [03 — Packages & visibility](03-packages-and-visibility.md).

```go
type User struct {
	ID    int      // exported — usable from other packages
	Name  string   // exported
	email string   // unexported — invisible outside this package
}
```

**This is not a styling choice.** The video shows what happens by lowercasing one field of a fider struct that another package imports. Errors appear immediately everywhere that field was used.

It matters for JSON too. `encoding/json` can only marshal and unmarshal **exported** fields. A lowercase field will silently disappear from your JSON output.

### Real-world shape (fider)

```go
type User struct {
	ID        int             `json:"id"`
	Name      string          `json:"name"`
	Tenant    *Tenant         `json:"-"`
	Email     string          `json:"-"`
	Role      Role            `json:"role"`
	Providers []*UserProvider `json:"-"`
}
```

Structs hold anything. Ints, strings, bools, slices, maps, pointers, and other structs. That flexibility is why they are everywhere.

---

**Next:** [08 — Arrays & slices](08-arrays-and-slices.md)
