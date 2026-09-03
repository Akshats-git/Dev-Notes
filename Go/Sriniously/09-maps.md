# 09 — Maps

[▶ 3:19:30](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=11970s)

A map is Go's hash table. Other languages call this a dictionary, a hash, or a hashmap. In Go it is a `map`.

---

## The shape

```go
map[KeyType]ValueType
```

Three rules.

**Keys must be comparable.** They have to work with `==`. That includes strings, numbers, bools, pointers, and structs made of comparable fields. It excludes slices, maps, and functions.

**Values can be anything.**

Keys are usually strings. They do not have to be.

---

## Why maps exist: fast lookup

> "We use maps when we want to find things quickly."

Maps give you O(1) average lookup, insert, and delete.

That is the whole reason to use one. If you are storing a record, use a struct. If you are going to search for things by key, use a map.

---

## Creating a map

### With `make`

A map is a reference type, so `make` works on it.

```go
type Vertex struct {
	Lat, Long float64
}

var m map[string]Vertex        // declared — value is nil

func main() {
	m = make(map[string]Vertex)
	m["Bell Labs"] = Vertex{40.68433, -74.39967}
	fmt.Println(m["Bell Labs"])   // {40.68433 -74.39967}
}
```

### With a map literal

```go
var m = map[string]Vertex{
	"Bell Labs": {40.68433, -74.39967},
	"Google":    {37.42202, -122.08408},
}
```

Since Go already knows the value type, you can leave out `Vertex` on each entry.

---

## ⚠️ The `var` trap

Writing to a nil map panics.

```go
var m map[string]int
m["one"] = 9        // 💥 panic: assignment to entry in nil map
```

`var` gives you the zero value of a map. The zero value is `nil`. **You cannot write to a nil map.**

> **There are exactly two ways to create a usable map. Use `make`, or use a map literal.** Never use `var` unless you know exactly what you are doing.

```go
m := make(map[string]int)        // ✅
m := map[string]int{}            // ✅
```

*(Only writes panic. Reading from a nil map is safe and returns the zero value. Either way, "always use `make` or a literal" is the rule to internalize.)*

---

## Changing a map

```go
m := make(map[string]int)

m["Answer"] = 42               // insert
fmt.Println(m["Answer"])       // 42

m["Answer"] = 48               // update
fmt.Println(m["Answer"])       // 48

delete(m, "Answer")            // delete — built-in function
fmt.Println(m["Answer"])       // 0  ← ???
```

---

## The comma-ok idiom

That last `0` is a problem. **Is the value really 0, or is the key missing?**

A missing key returns the zero value of the value type. So you cannot tell the two apart from the value alone.

Go's answer is the two-value form of a map read.

```go
v, ok := m["Answer"]
```

| Return | Meaning |
|---|---|
| `v` | the value if the key exists, otherwise the zero value |
| `ok` | `true` if the key exists, `false` if it does not |

```go
if v, ok := m["Answer"]; ok {
	fmt.Println("present, value is", v)
} else {
	fmt.Println("key not set")
}
```

**By convention you name the second variable `ok`.** Other engineers will immediately see that you are testing for existence.

This combines well with `if`'s init statement. See [06 — Control flow](06-control-flow.md).

---

## Looping over a map

You use the same `range` as with slices. But it gives you **key and value** instead of index and value.

```go
for key, value := range m {
	fmt.Println(key, value)
}

for _, value := range m { ... }   // keys discarded
for key := range m { ... }        // values omitted
```

### ⚠️ The order is randomized on purpose

Go **deliberately randomizes** map iteration order. This stops you from accidentally depending on it. Run the same program twice and you get different orders.

**If you need a stable order, pull the keys into a slice and sort them yourself.**

---

## Maps vs structs

[▶ 3:30:40](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=12640s)

Here is the one-line rule.

> **Structs are for storing data. Maps are for looking data up.**

That is oversimplified, but it is a good default. In backend code you want a struct roughly 70% of the time.

### The real differences

| | Struct | Map |
|---|---|---|
| **Fields known at compile time?** | Yes. Fixed and static. | No. Dynamic, arbitrary keys. |
| **Can you `range` over it?** | ❌ No. The fields are already written in your code. There is nothing to discover. | ✅ Yes |
| **Do entries have stable memory addresses?** | ✅ Yes. You can take `&s.Field`. | ❌ **No** |
| **Iteration order** | not applicable | randomized on purpose |

**Why map entries have no fixed address.** Maps use hashes and buckets internally. They rehash as they grow, which physically moves entries. So `&m["key"]` is illegal. The address would not stay valid.

Struct fields sit at fixed offsets, so they are addressable.

---

## Real-world check (fider / incubator-answer)

[▶ 3:33:10](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=12790s)

There is a clear theme. **Maps are for data whose shape you do not know at compile time.**

**HTTP headers** use `map[string]string`. You cannot list every header a client might send, so a struct is impossible.

**Arbitrary JSON** goes into `map[string]interface{}` when you do not know the keys ahead of time. If you *do* know them, use a struct with tags instead. See [07 — Pointers & structs](07-pointers-and-structs.md).

The decision usually makes itself. **Known fixed fields means struct. Unknown keys, or a need for fast lookup, means map.**

---

**Next:** [10 — Functions as values](10-functions-as-values.md)
