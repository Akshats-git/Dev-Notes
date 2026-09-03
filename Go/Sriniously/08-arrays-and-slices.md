# 08 — Arrays & Slices

This is the longest section of the video, about 55 minutes. It is also the most important data structure in day-to-day Go.

---

## Arrays

[▶ 2:22:34](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=8554s)

An array is a collection of **homogeneous** data. Every element has the same type.

```go
var a [2]string
a[0] = "Hello"
a[1] = "World"
fmt.Println(a[0], a[1])   // Hello World
fmt.Println(a)            // [Hello World]

primes := [6]int{2, 3, 5, 7, 11, 13}
```

### The one thing to remember

**The size is part of the type.**

`[2]string` and `[3]string` are **different types**. Go is statically typed. So this has a big consequence.

> **Arrays cannot be resized.** You cannot grow one. You cannot shrink one. You cannot append to one. The length you declare is the length forever.

That is why arrays are rare in real Go code. You use one when you know a fixed count up front. The rest of the time you use a **slice**.

---

## Slices

[▶ 2:25:34](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=9334s)

A slice is a dynamically-sized view into an array.

It is a **reference type**. It does not hold data. It points at a **backing array**.

### Making a slice from an array

```go
primes := [6]int{2, 3, 5, 7, 11, 13}
s := primes[1:4]
fmt.Println(s)   // [3 5 7]
```

The syntax is `[low:high]`. **Low is included. High is excluded.** So `[1:4]` gives you indices 1, 2, and 3.

---

## What a slice actually is

This model explains everything else. A slice is a small struct with three fields.

```
┌──────────────────────────────────┐
│ SLICE                            │
│   ptr  → first element it covers │
│   len  → number of elements      │
│   cap  → from ptr to the END of  │
│          the backing array       │
└──────────────────────────────────┘
             │
             ▼
   backing array: [ 2 | 3 | 5 | 7 | 11 | 13 ]
                    0   1   2   3    4    5
```

| Field | What it means |
|---|---|
| **pointer** | the address of the first element the slice covers |
| **length** | how many elements the slice currently shows you |
| **capacity** | the count from the slice's first element **to the end of the backing array** |

That last one is the definition people get wrong. Capacity is measured against the **backing array**, not against the slice.

---

## Slices share their backing array

This is the biggest gotcha.

```go
names := [4]string{"John", "Paul", "George", "Ringo"}

a := names[0:2]   // len 2, cap 4, points at "John"
b := names[1:3]   // len 2, cap 3, points at "Paul"

fmt.Println(a, b)   // [John Paul] [Paul George]

b[0] = "XXX"        // mutate through b

fmt.Println(a, b)   // [John XXX] [XXX George]
fmt.Println(names)  // [John XXX George Ringo]
```

**One write through `b` changed `a` and `names` too.** All three view the same storage underneath.

This is what "reference type" means in practice. Change one element and every slice sharing that backing array sees it. It is the single biggest source of surprise bugs with slices.

---

## Slice literals

You do not need an array first. You can declare a slice directly. Go creates the backing array for you.

```go
q := []int{2, 3, 5, 7, 11, 13}
r := []bool{true, false, true, true, false, true}

s := []struct {
	i int
	b bool
}{
	{2, true},
	{3, false},
	{5, true},
}
```

**Here is how to tell an array from a slice at a glance.** An array type has a number in the brackets, like `[6]int`. A slice has empty brackets, like `[]int`.

For a slice literal, `len` and `cap` both equal the number of elements.

---

## Slice bounds are optional

```go
s := []int{2, 3, 5, 7, 11, 13}

s = s[1:4]   // [3 5 7]
s = s[:2]    // low defaults to 0
s = s[1:]    // high defaults to len(s)
s = s[:]     // both — the whole thing
```

You can also slice **an existing slice**, not just an array.

---

## `len` and `cap` in action

```go
func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}

s := []int{2, 3, 5, 7, 11, 13}
printSlice(s)     // len=6 cap=6 [2 3 5 7 11 13]

s = s[:0]         // slice it down to zero length
printSlice(s)     // len=0 cap=6 []

s = s[:4]         // extend it back out — capacity allows it
printSlice(s)     // len=4 cap=6 [2 3 5 7]

s = s[2:]         // drop the first two
printSlice(s)     // len=2 cap=4 [5 7]
```

### Reading that trace

`s[:0]` gives length 0, but **capacity stays 6**. The backing array is untouched. You only narrowed the window. That is why you can widen it again.

`s[:4]` widens it back to length 4. This is legal because capacity is at least 4.

`s[2:]` moves the **start pointer** forward by 2. So capacity drops from 6 to 4.

### The capacity rule

> **Capacity only shrinks when you re-slice starting from an index above 0.**

Slicing off the end with `s[:n]` does not touch capacity. That is because capacity is measured from the start pointer to the end of the backing array.

---

## The zero value of a slice is `nil`

```go
var s []int
printSlice(s)             // len=0 cap=0 []
if s == nil {
	fmt.Println("nil!")   // prints
}
```

A slice declared with `var` is `nil`. Its length is 0 and its capacity is 0.

It prints as `[]`, but it really is `nil`. Comparing to `nil` is how you check.

Unlike a nil map, **a nil slice is safe to `append` to.** `append` does the allocation for you.

---

## Creating slices with `make`

`make` is a built-in function. It initializes **reference types**: slices, maps, and channels.

```go
make([]int, 5)      // type, length          → cap defaults to len = 5
make([]int, 0, 5)   // type, length, capacity → len 0, cap 5
```

| Argument | Meaning |
|---|---|
| 1st | the type, like `[]int`. It must be explicit, since `make` also builds maps and channels. |
| 2nd | length |
| 3rd (optional) | capacity. Defaults to the length. |

**Capacity can never be less than length.** That is an error.

### Worked example

```go
a := make([]int, 5)      // len=5 cap=5 [0 0 0 0 0]
b := make([]int, 0, 5)   // len=0 cap=5 []
c := b[:2]               // len=2 cap=5 [0 0]
d := c[2:5]              // len=3 cap=3 [0 0 0]
```

`a` has length 5. So it has five slots, each holding the zero value of `int`.

`b` has length 0, so it prints empty. But it has room for 5.

`c` was sliced from index 0. So capacity stays at 5.

`d` was sliced starting at index 2. So capacity drops from 5 to 3. Notice you can slice up to `cap`, not just up to `len`.

---

## Two-dimensional slices

```go
board := [][]string{
	{"_", "_", "_"},
	{"_", "_", "_"},
	{"_", "_", "_"},
}

board[0][0] = "X"
board[2][2] = "O"
```

This is a slice whose elements are also slices. It works exactly the way you would expect.

---

## `append`

[▶ 2:59:15](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=10755s)

```go
var s []int
s = append(s, 0)          // len=1 cap=1 [0]
s = append(s, 1)          // len=2 cap=2 [0 1]
s = append(s, 2, 3, 4)    // len=5 cap=6 [0 1 2 3 4]
```

Two rules here.

**`append` is variadic.** It takes the slice first, then any number of elements.

**You must reassign the result.** Always write `s = append(s, x)`. `append` may allocate a new backing array, so it returns a slice that might be different from the one you passed in.

### What `append` does internally

1. It checks whether the backing array has spare room, meaning `len < cap`.
2. **If there is room,** it writes into the existing array and bumps the length. This is cheap.
3. **If there is no room,** it allocates a **new, larger backing array**, copies everything over, and points the returned slice at it. This is expensive.

### How capacity grows

You can watch it happen:

```go
var s []int
for i := 0; i < 20; i++ {
	s = append(s, i)
	printSlice(s)
}
```

```
len=1  cap=1
len=2  cap=2
len=3  cap=4     ← doubled
len=4  cap=4
len=5  cap=8     ← doubled
len=6  cap=8
len=7  cap=8
len=8  cap=8
len=9  cap=16    ← doubled
...
```

**The pattern is simple. Whenever `len == cap`, the next append doubles the capacity** and reallocates.

The doubling continues up to about 256 elements. After that Go grows by roughly **1.25×** instead of 2×. There is a `nextslicecap` function in the runtime source that implements this. It is an internal detail. The point is that growth is amortized, not linear.

---

## The one optimization worth applying

Every reallocation costs you. Go has to find new memory, allocate a bigger array, and copy every element.

If you know the final size ahead of time, you can skip all of that.

```go
// ❌ 3 reallocations to reach 5 elements: cap 0→1→2→4→8
var s []int
for i := 0; i < 5; i++ {
	s = append(s, i)
}
```

```go
// ✅ zero reallocations: capacity is right from the start
s := make([]int, 0, 5)
for i := 0; i < 5; i++ {
	s = append(s, i)
}
// cap stays 5 through every iteration
```

**Look carefully at `make([]int, 0, 5)`.** Length 0, capacity 5.

Do not write `make([]int, 5, 5)`. That pre-fills five zeros, and your appends would go *after* them.

> **Rule of thumb.** If you know the maximum number of elements ahead of time, preallocate with `make([]T, 0, n)`. This applies to database result sets and to transforming an existing slice.

This is the only practical reason to understand capacity. Otherwise you will never think about it. `append` handles everything.

---

## `range` — looping over slices

[▶ 3:17:33](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=11853s)

The traditional loop works:

```go
for i := 0; i < len(pow); i++ {
	fmt.Printf("%d = %d\n", i, pow[i])
}
```

But `range` is the idiomatic way:

```go
for i, v := range pow {
	fmt.Printf("2**%d = %d\n", i, v)
}
```

Each pass gives you **two values. The index, and a *copy* of the element.**

> ⚠️ **`v` is a copy.** Assigning to `v` inside the loop does **not** change the slice. To change it, index directly with `pow[i] = ...`.

### Skipping a value

```go
for _, v := range pow { ... }   // index discarded
for i := range pow { ... }      // value omitted entirely
```

`_` is the blank identifier. It means "there is a value here and I do not want it."

---

## Real-world check (fider)

[▶ 3:18:40](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=11920s)

All three patterns show up constantly:

```go
tags := []string{"bug", "feature"}          // slice literal
titles := make([]string, 31)                 // make with known size
for _, title := range titles { ... }         // range, index discarded
```

Slices are one of the most-used types in any Go codebase.

---

**Next:** [09 — Maps](09-maps.md)
