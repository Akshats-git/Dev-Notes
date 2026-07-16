# JavaScript Foundation — Complete Guide

> Based on the code in `code+files/jsFoundation/` — every concept explained with context, not just syntax.

---

## Table of Contents

1. [Output & Printing](#1-output--printing)
2. [Data Types](#2-data-types)
3. [Variables](#3-variables)
4. [Operators](#4-operators)
5. [Primitives vs Non-Primitives](#5-primitives-vs-non-primitives)
6. [Conditionals](#6-conditionals)
7. [Arrays](#7-arrays)
8. [Loops](#8-loops)
9. [Functions](#9-functions)
10. [Object-Oriented Programming](#10-object-oriented-programming)
11. [Prototypes](#11-prototypes)
12. [DOM Manipulation & Events](#12-dom-manipulation--events) (incl. [BOM](#bom--browser-object-model))
13. [The Event Loop](#13-the-event-loop)
14. [Closures](#14-closures)
15. [`this` Context, bind / call / apply](#15-this-context-bind--call--apply)
16. [Promises](#16-promises)
17. [Async / Await](#17-async--await)
18. [Generators & Iterators](#18-generators--iterators)
19. [Modules (CJS & ESM)](#19-modules-cjs--esm)

---

## 1. Output & Printing

**Source:** `part1/printing.js`

Before writing logic, you need a way to see what your code is doing. In a browser, `console.log` prints to the DevTools console. In Node.js it prints to the terminal. This is your primary debugging tool — you'll use it constantly.

`console.table` is useful when you want to inspect an object or array in a readable grid format instead of the compressed `{key: value}` format. `console.warn` is the same as `console.log` but shows a yellow warning icon — helpful for flagging potential issues without crashing anything. `process.stdout.write` is Node.js-only and writes directly to the terminal without adding a newline at the end (unlike `console.log` which always adds one).

```js
console.log("Hello");                   // prints: Hello
console.log("chai");                    // each call starts on a new line

process.stdout.write("chai");           // no newline at end
process.stdout.write("chai");           // prints: chaichai (runs together)

console.table({ city: "Jaipur" });      // prints a nice table in the terminal
console.warn({ city: "Jaipur" });       // same output but with a warning icon
```

| Method | What it does |
|---|---|
| `console.log()` | The everyday tool — prints anything to the console |
| `console.table()` | Shows objects/arrays as a formatted table — great for readability |
| `console.warn()` | Yellow warning in DevTools — same output, different visual signal |
| `console.error()` | Red error message — use when something actually went wrong |
| `process.stdout.write()` | Node.js only — writes without appending a newline |

---

## 2. Data Types

**Source:** `part1/datatype.js`

Every value in JavaScript has a type. The type determines what operations you can perform on it — you can do math on numbers, search inside strings, and loop over arrays. JavaScript figures out types automatically at runtime (it's dynamically typed), so you don't have to declare what type a variable will hold.

There are two categories: **primitive** types (simple, single values) and **non-primitive** types (complex, structured data).

```
┌─────────────────────────────────────────────┐
│               JavaScript Types              │
│                                             │
│   Primitive            Non-Primitive        │
│   ──────────           ─────────────        │
│   String               Object               │
│   Number               Array                │
│   Boolean              Function             │
│   BigInt               Date                 │
│   undefined                                 │
│   null (special case)                       │
│   Symbol                                    │
└─────────────────────────────────────────────┘
```

Here's what each one looks like in code:

```js
let score      = 102;                         // Number
let name       = "chaicode.com";              // String
let isLoggedin = false;                       // Boolean
let teaTypes   = ["lemon tea", "oolong tea"]; // Array — a type of Object
let user       = { firstname: "hitesh" };     // Object
```

### Checking the type of a value

The `typeof` operator returns a string describing the type of any value. You'll use this in conditionals to make sure a value is what you expect before operating on it.

```js
typeof 42           // "number"
typeof "hello"      // "string"
typeof true         // "boolean"
typeof undefined    // "undefined"
typeof null         // "object"   ← famous JS bug — null is NOT an object
typeof Symbol()     // "symbol"
typeof {}           // "object"
typeof []           // "object"   ← arrays are objects under the hood
typeof function(){} // "function"
```

The `typeof null === "object"` result is a well-known historical mistake in JavaScript. The language can't fix it now because it would break millions of websites that rely on that behaviour. To check for `null` specifically, use `=== null` directly.

---

## 3. Variables

**Source:** `part1/changes.js`, `part1/datatype.js`

A variable is a named container for a value. You can create one, put a value in it, and later read or change that value. JavaScript has three keywords for declaring variables: `var`, `let`, and `const`. They differ in how widely they're visible (scope) and whether you can reassign them.

```js
var   score = 102;   // old way — function-scoped, has quirky hoisting behaviour
let   score = 102;   // modern — block-scoped, can be reassigned
const PI    = 3.14;  // modern — block-scoped, cannot be reassigned
```

**Scope** is the region of code where a variable exists. `var` leaks out of `if` blocks and `for` loops — it only respects function boundaries. `let` and `const` are block-scoped, meaning they only live inside the `{ }` they were declared in. This makes your code much more predictable.

```
Scope diagram:

  Global scope
  ┌──────────────────────────────────────┐
  │  var → visible to the whole function │
  │                                      │
  │  Function scope                      │
  │  ┌──────────────────────────┐        │
  │  │  let / const are visible │        │
  │  │  ┌────────────────────┐  │        │
  │  │  │  Block scope { }   │  │        │
  │  │  │  let / const live  │  │        │
  │  │  │  only here         │  │        │
  │  │  └────────────────────┘  │        │
  │  └──────────────────────────┘        │
  └──────────────────────────────────────┘
```

### const does not mean immutable

`const` prevents you from reassigning the variable to a completely new value. But if the value is an object or array, you can still change what's *inside* it — the variable itself just always points to the same object.

```js
// Reassignment — always blocked by const
const username = "hiteshdotcom";
username = "hitesh"; // ❌ TypeError: Assignment to constant variable

// Mutation — allowed even with const
const user = { name: "hitesh" };
user.name = "chai";  // ✅ works fine — same object, changed property
user = {};           // ❌ not allowed — would be a reassignment
```

In practice: use `const` by default. Switch to `let` only when you know you'll reassign the value (like a loop counter). Avoid `var` entirely in modern code.

---

## 4. Operators

**Source:** `part1/operations.js`, `part1/assignment.js`, `part1/logical.js`, `part1/opePre.js`

Operators are symbols that tell JavaScript to do something with values. There are several categories.

### Arithmetic Operators

These perform math. The modulo `%` operator returns the **remainder** after division — very useful for checking if a number is even/odd, or cycling through a list.

```js
let addition  = 4 + 5;   // 9
let subtract  = 9 - 3;   // 6
let mult      = 3 * 5;   // 15
let divi      = 8 / 2;   // 4
let reminder  = 9 % 2;   // 1  — 9 divided by 2 leaves remainder 1
let expo      = 2 ** 3;  // 8  — 2 to the power of 3
```

### Increment & Decrement

These are shortcuts for adding or subtracting 1. You'll see them constantly in loop counters.

```js
let myscore = 110;
myscore++;   // same as: myscore = myscore + 1   → 111

let credits = 56;
credits--;   // same as: credits = credits - 1   → 55
```

Post-increment (`x++`) uses the current value first, then increments. Pre-increment (`++x`) increments first, then returns the new value. For simple counters this distinction rarely matters, but it's good to know.

### Comparison Operators

These compare two values and return `true` or `false`. The difference between `==` and `===` is critical.

`==` does **type coercion** — it tries to convert both sides to the same type before comparing. This leads to surprising results like `0 == false` being `true`. `===` checks both the value AND the type, with no conversion. Always prefer `===`.

```js
let num1 = 3, num2 = 3, num3 = 6;

console.log(num1 == num2);   // true  — same value
console.log(num1 != num3);   // true  — different values
console.log(num1 > num3);    // false — 3 is not greater than 6
console.log(num1 < num3);    // true  — 3 is less than 6

// Type coercion trap:
console.log(0 == false);     // true  ← dangerous loose equality
console.log(0 === false);    // false ← correct strict equality
```

### Assignment Operators

Shorthand for updating a variable based on its current value. `num1 /= 5` is just a shorter way to write `num1 = num1 / 5`.

```js
let num1 = 10;
num1 /= 5;    // num1 is now 2
num1 += 5;    // num1 is now 7
num1 -= 3;    // num1 is now 4
num1 *= 2;    // num1 is now 8
```

### Logical Operators

Used to combine or invert boolean conditions. Most commonly seen in `if` statements.

- `&&` (AND) — both sides must be truthy for the result to be truthy
- `||` (OR) — at least one side must be truthy
- `!` (NOT) — flips a truthy value to false, or a falsy value to true

```js
let isLoggedin = true;
let isPaid     = true;
let isEmailUser  = true;
let isGoogleUser = false;

console.log(isLoggedin && isPaid);       // true  — both true
console.log(isEmailUser || isGoogleUser); // true  — at least one true
console.log(!isLoggedin);                // false — flips true to false
```

A common real-world use: showing gated content — `if (isLoggedin && isPaid)` means the user must be both logged in AND have a paid account.

### Operator Precedence

Just like in maths, multiplication happens before addition. Parentheses always run first and let you be explicit about the order of operations.

```js
// Without parentheses: multiplication before addition
let a = 2 + 3 * 2;    // 8  (not 10 — * runs before +)

// With parentheses: force the order you want
let score = ((2 * (3 + 2)) - 1);
//            ─────────── inner parentheses first: 3+2 = 5
//           ───────────── then 2*5 = 10
//          ──────────────── then 10-1 = 9
console.log(score); // 9
```

---

## 5. Primitives vs Non-Primitives

**Source:** `part2/primitives.js`, `part2/nonPrimitives.js`

This is one of the most important concepts in JavaScript. It affects how values are stored, copied, and compared. Primitive types are simple values stored directly. Non-primitive types are objects stored in memory, and variables just hold a *reference* (address) to that memory.

### Primitive Types — stored by value

When you copy a primitive, you get a completely independent copy. Changing one has no effect on the other.

```js
let balance  = 120;           // Number
let isActive = true;          // Boolean
let firstname = null;         // null — intentionally set to "nothing"
let lastname  = undefined;    // undefined — declared but never given a value

// Symbols — every Symbol() call creates a unique identifier
let sm1 = Symbol("hitesh");
let sm2 = Symbol("hitesh");
console.log(sm1 === sm2); // false — even with identical labels, they're different
```

`null` vs `undefined` is a common source of confusion. Think of it this way: `undefined` means "this variable exists but nobody has given it a value yet." `null` means "this variable exists and I'm explicitly saying it's empty." When you create a user object but haven't loaded their profile yet, setting `profile = null` is intentional, whereas `profile = undefined` usually means you forgot to set it.

### Template Literals

Template literals use backticks `` ` `` instead of quotes. They let you embed variables and expressions directly inside a string using `${}`. This is much cleaner than concatenating strings with `+`.

```js
let username     = "hitesh";

// Old way — concatenation with +
let oldGreet = "Hello " + username + "!";   // messy with multiple variables

// Modern way — template literal
let greetMessage = `Hello ${username}!`;     // clean, readable
let demoOne      = `Value is ${2 * 2}`;      // can run code inside ${}
// → "Value is 4"
```

### Non-Primitive Types — stored by reference

Objects, arrays, and dates are not stored directly in the variable. The variable holds a *reference* — think of it as a pointer to where the actual data lives in memory. This is why copying an object and then modifying the copy also modifies the original.

```js
const username = {
  "first name": "hitesh",   // keys with spaces must use quotes
  isLoggedin: true,
};

// You can freely add new properties even on a const object
username.lastname = "choudhary"; // ✅ this is mutation, not reassignment

// Access using dot notation or bracket notation
console.log(username.isLoggedin);      // true
console.log(username["first name"]);   // "hitesh" — bracket is required for spaced keys

let today = new Date();        // Date is a built-in object
today.getDate();               // returns today's day of the month (e.g. 15)
```

### Type Conversion

JavaScript can convert between types explicitly using functions like `Number()`. The results are sometimes surprising.

```js
Number("44")     // 44   — valid number string → number
Number("2abc")   // NaN  — "not a number" — invalid conversion
Number(null)     // 0    — null becomes 0
Number(true)     // 1
Number(false)    // 0
```

`NaN` stands for "Not a Number" but its type is actually `"number"` — another JS quirk. It's what you get when a numeric operation produces an invalid result. Any arithmetic with `NaN` stays `NaN` (e.g. `NaN + 5 = NaN`).

### The fundamental difference — value vs reference

```
Memory model:

  Primitive (copy by value)          Non-Primitive (copy by reference)
  ─────────────────────────          ──────────────────────────────────
  let a = 5;                         let arr = [1, 2, 3];
  let b = a;   ← b gets a COPY       let copy = arr;  ← copy points to SAME array
  b = 10;                            copy.push(4);
  // a is still 5 — untouched        // arr is now [1, 2, 3, 4]
                                     // changing copy changed arr too!
```

This distinction directly affects how you write functions, handle state, and copy data — you'll see it come up constantly in arrays and objects.

---

## 6. Conditionals

**Source:** `part3/conditon-challenges.js`

Conditionals let your program make decisions. The code inside an `if` block only runs if the condition evaluates to `true`. If it's `false`, JavaScript skips it and either runs the `else` block (if there is one) or does nothing.

```js
let num1 = 5;
let num2 = 8;

if (num1 > num2) {
  console.log("num1 is greater than num2");
} else {
  console.log("Nope, num1 is NOT greater");
}
// → "Nope, num1 is NOT greater" — because 5 is not greater than 8
```

You can also check the type of a value before working with it. This is important because JavaScript doesn't enforce types, so a function might receive a string when it expected a number.

```js
let score = "44";   // this is a string, not a number

if (typeof score === "number") {
  console.log("Yep, this is a number");
} else {
  console.log("No that is not a number");
}
// → "No that is not a number"
```

### Truthy and Falsy

In JavaScript, every value is either *truthy* or *falsy*. When you put a value directly in an `if` condition (not a comparison, just the value), JavaScript automatically converts it to a boolean. This is called an implicit boolean conversion.

**Falsy values — these evaluate to `false` in a condition:**

| Value | Reason |
|---|---|
| `false` | obviously false |
| `0` | zero |
| `""` | empty string |
| `null` | intentional emptiness |
| `undefined` | unset variable |
| `NaN` | invalid number |

Everything else is **truthy** — including `"0"` (a string), `[]` (empty array), and `{}` (empty object). This trips up a lot of beginners.

```js
let isTeaReady = false;

if (isTeaReady) {
  console.log("Tea is Ready");
} else {
  console.log("Tea is NOT ready");
}
// → "Tea is NOT ready" — because false is falsy

// A practical example — checking if an array has items
let items = ["item1"];

if (items.length === 0) {
  console.log("Array is empty");
} else {
  console.log("Array is NOT empty");
}
// → "Array is NOT empty" — items.length is 1, which is not 0
```

Truthy/falsy is useful for concise guards: `if (username) { ... }` will only run if `username` is a non-empty string — no need to write `username !== null && username !== undefined && username !== ""`.

---

## 7. Arrays

**Source:** `part4/arrayChallenges.js`

An array is an ordered list of values. Each value has a position called an *index*, and indexes start at 0, not 1. So the first item is at index `0`, the second at `1`, and so on.

```js
let teaFlavors = ["green tea", "black tea", "oolong tea"];
//                    [0]           [1]           [2]

let firstTea    = teaFlavors[0];  // "green tea"
let favoriteCity = teaFlavors[2]; // "oolong tea" — third item, index 2
```

Arrays can hold any mix of values — strings, numbers, booleans, objects, even other arrays. The `.length` property tells you how many items are in the array.

### Adding and Removing Items

`push` adds to the end. `pop` removes from the end and returns the removed value. These are the most common operations.

```js
let citiesVisited = ["Mumbai", "Sydney"];
citiesVisited.push("Berlin");
// → ["Mumbai", "Sydney", "Berlin"]

let teaOrders = ["chai", "iced tea", "matcha", "earl grey"];
const lastOrder = teaOrders.pop();
// lastOrder = "earl grey", teaOrders = ["chai", "iced tea", "matcha"]
```

### Checking Membership

`includes()` returns `true` if the value is in the array, `false` if not. It uses strict equality (`===`) internally.

```js
let cityBucketList = ["Kyoto", "London", "Cape Town", "Vancouver"];
let isLondonInList = cityBucketList.includes("London");  // true
```

### Soft Copy vs Hard Copy

This is where the "reference vs value" concept from section 5 shows up directly. When you do `let b = a` for an array, both variables point to the *same* array in memory. Modifying one affects the other. This is a **soft copy** (reference copy). If you want an independent copy that doesn't affect the original, you need a **hard copy**.

```js
// Soft copy — same reference, changes affect both
let popularTeas    = ["green tea", "oolong tea", "chai"];
let softCopyTeas   = popularTeas;
popularTeas.pop();
console.log(softCopyTeas); // ["green tea", "oolong tea"] — changed too!

// Hard copy — independent, changes don't affect original
let topCities      = ["Berlin", "Singapore", "New York"];
let hardCopyCities = [...topCities];   // spread operator creates a new array
topCities.pop();
console.log(hardCopyCities); // ["Berlin", "Singapore", "New York"] — unchanged
```

```
Soft Copy vs Hard Copy:

  Soft copy (same reference):        Hard copy (new reference):
  ──────────────────────────         ─────────────────────────
  let a = [1, 2, 3];                 let a = [1, 2, 3];
  let b = a;                         let b = [...a];
       │                                  │
       └──► [1, 2, 3] ◄── both           ├──► [1, 2, 3]  ← original
                                          └──► [1, 2, 3]  ← new copy
  Mutating b mutates a!              Mutating b does NOT affect a
```

### slice — extract a portion without modifying the original

`slice(start, end)` returns a new array containing items from index `start` up to (but **not including**) index `end`. The original array is never touched. If you omit `end`, it goes all the way to the last element.

```js
let teas = ["green tea", "black tea", "oolong tea", "chai", "masala"];

teas.slice(1, 3);  // → ["black tea", "oolong tea"]
//                         index 1        index 2    (index 3 is excluded)

teas.slice(2);     // → ["oolong tea", "chai", "masala"]  (from index 2 to end)
teas.slice(-2);    // → ["chai", "masala"]  (negative index = count from the end)

console.log(teas); // → original is untouched
```

`slice` with no arguments is one of the standard ways to make a **hard copy** of an array:

```js
let hardCopy = teas.slice();  // copy of the whole array
```

```
slice visualised on ["green tea", "black tea", "oolong tea", "chai"]:

   index:    0            1            2            3
          ┌──────────┬──────────┬───────────┬──────┐
          │ green tea│ black tea│ oolong tea│ chai │
          └──────────┴──────────┴───────────┴──────┘
                     ↑                      ↑
              slice(1, 3) starts here  ends before here
              result → ["black tea", "oolong tea"]
```

### splice — add, remove, or replace items in place

`splice(start, deleteCount, ...itemsToInsert)` **mutates the original array**. It removes `deleteCount` elements starting at `start`, and optionally inserts new elements in their place. It returns an array of the deleted elements.

```js
let cities = ["London", "Paris", "Tokyo", "Berlin", "Sydney"];

// Remove 2 elements starting at index 1
let removed = cities.splice(1, 2);
// removed → ["Paris", "Tokyo"]
// cities  → ["London", "Berlin", "Sydney"]  ← original is changed

// Insert without removing (deleteCount = 0)
cities.splice(1, 0, "Mumbai", "Dubai");
// cities → ["London", "Mumbai", "Dubai", "Berlin", "Sydney"]

// Replace 1 element at index 2
cities.splice(2, 1, "Cairo");
// cities → ["London", "Mumbai", "Cairo", "Berlin", "Sydney"]
```

```
splice vs slice — the one-line distinction:

  slice  →  NON-destructive — returns a new array, original untouched
  splice →  DESTRUCTIVE      — mutates the original, returns what was removed
```

A good memory trick: **sl-I-ce** = non-destructive (i = isolated copy), **sp-L-ice** = in-pLace mutation.

### shift & unshift — add/remove from the front

`push`/`pop` work on the **end** of an array. `unshift`/`shift` do the same thing but at the **beginning**. They're slower than push/pop on large arrays because every existing element has to be re-indexed, but for small arrays it doesn't matter.

```js
let queue = ["first", "second", "third"];

// shift — remove and return the first element
let served = queue.shift();
// served → "first"
// queue  → ["second", "third"]

// unshift — add one or more elements to the front
queue.unshift("new first", "new second");
// queue → ["new first", "new second", "second", "third"]
```

```
push / pop  vs  unshift / shift:

  unshift("x") ──►  [ x | a | b | c ]  ◄── push("x")
  shift()      ◄──  [ a | b | c | x ]  ──► pop()
                  removes from front        removes from end
```

### indexOf & lastIndexOf — find the position of a value

`indexOf` returns the index of the **first** occurrence of a value, or `-1` if it isn't found. `lastIndexOf` starts searching from the end. Use these when you need the *position*, not just whether the value exists.

```js
let teas = ["chai", "green tea", "chai", "oolong"];

teas.indexOf("chai");        // 0  — first occurrence
teas.lastIndexOf("chai");    // 2  — last occurrence
teas.indexOf("black tea");   // -1 — not found

// Common pattern: check if something exists
if (teas.indexOf("chai") !== -1) {
  console.log("chai is in the list");
}
// (includes() is cleaner for existence checks; indexOf is for when you need the position)
```

### find & findIndex — search with a condition

Unlike `indexOf` (which matches by exact value), `find` and `findIndex` accept a callback and return the **first element/index** that satisfies your condition. Essential when searching arrays of objects.

```js
let users = [
  { id: 1, name: "Hitesh" },
  { id: 2, name: "Chai"   },
  { id: 3, name: "Alice"  },
];

// find — returns the element itself, or undefined
let user = users.find((u) => u.id === 2);
// → { id: 2, name: "Chai" }

// findIndex — returns the index, or -1
let idx = users.findIndex((u) => u.name === "Alice");
// → 2
```

### map — transform every element into something new

`map` runs a callback on each element and returns a **new array** of the results. The original array is unchanged. This is one of the most used array methods in JavaScript — you'll use it constantly to transform data.

```js
let prices    = [100, 200, 300];
let discounted = prices.map((price) => price * 0.9);
// → [90, 180, 270]  — original prices array untouched

let teas = ["green tea", "chai", "oolong"];
let upper = teas.map((tea) => tea.toUpperCase());
// → ["GREEN TEA", "CHAI", "OOLONG"]
```

### filter — keep only elements that pass a test

`filter` runs a callback on each element and returns a **new array** containing only the elements for which the callback returned `true`. Think of it as picking items from a list based on a rule.

```js
let numbers = [1, 2, 3, 4, 5, 6, 7, 8];

let evens = numbers.filter((n) => n % 2 === 0);
// → [2, 4, 6, 8]

let cities = [
  { name: "London",   pop: 8900000 },
  { name: "Paris",    pop: 2200000 },
  { name: "Tokyo",    pop: 9000000 },
];

let largeCities = cities.filter((city) => city.pop > 3000000);
// → [{ name: "London", ... }, { name: "Tokyo", ... }]
```

### reduce — collapse an array into a single value

`reduce` iterates through the array, carrying an **accumulator** value that you update on each step. At the end, it returns the final accumulated value. It's the most flexible array method — you can implement map, filter, or anything else with it — but it takes a moment to get comfortable with.

```js
let numbers = [1, 2, 3, 4, 5];

// accumulator starts at 0, adds each number
let total = numbers.reduce((accumulator, current) => accumulator + current, 0);
// step 1: acc=0,  current=1  → 0+1  = 1
// step 2: acc=1,  current=2  → 1+2  = 3
// step 3: acc=3,  current=3  → 3+3  = 6
// step 4: acc=6,  current=4  → 6+4  = 10
// step 5: acc=10, current=5  → 10+5 = 15
// total → 15
```

The second argument to `reduce` (`0` above) is the **initial value** of the accumulator. Always provide it — without it, reduce uses the first element as the initial value, which can cause bugs.

### some & every — boolean checks across the array

`some` returns `true` if **at least one** element passes the test. `every` returns `true` only if **all** elements pass. Both stop early once they have their answer — `some` stops at the first `true`, `every` stops at the first `false`.

```js
let scores = [45, 72, 88, 31, 95];

scores.some((s) => s > 90);   // true  — 95 passes, stops there
scores.every((s) => s > 30);  // true  — all are above 30
scores.every((s) => s > 50);  // false — 45 and 31 fail
```

### sort — reorder elements in place

`sort` mutates the original array and reorders its elements. By default (no callback), it converts everything to strings and sorts alphabetically — this gives **wrong results for numbers**, so always pass a comparator function when sorting numbers.

```js
let teas = ["oolong", "chai", "green tea", "black tea"];
teas.sort();
// → ["black tea", "chai", "green tea", "oolong"]  (alphabetical ✅)

let scores = [10, 9, 100, 21, 3];
scores.sort();                         // → [10, 100, 21, 3, 9]  ❌ string sort!
scores.sort((a, b) => a - b);          // → [3, 9, 10, 21, 100]  ✅ numeric ascending
scores.sort((a, b) => b - a);          // → [100, 21, 10, 9, 3]  ✅ numeric descending
```

The comparator `(a, b) => a - b` works by the rule: negative = `a` comes first, positive = `b` comes first, zero = same order.

### join — turn an array into a string

`join` takes all elements, converts them to strings, and concatenates them with a separator you choose. The reverse of `"string".split(separator)`.

```js
let words = ["JavaScript", "is", "fun"];
words.join(" ");    // → "JavaScript is fun"
words.join(", ");   // → "JavaScript, is, fun"
words.join("-");    // → "JavaScript-is-fun"
words.join("");     // → "JavaScriptisfun"
```

### reverse — flip the array in place

`reverse` mutates the original array, reversing the order of its elements.

```js
let letters = ["a", "b", "c", "d"];
letters.reverse();
// → ["d", "c", "b", "a"]  — original array is now reversed

// Common pattern: sort then reverse for descending order without a comparator
["banana", "apple", "cherry"].sort().reverse();
// → ["cherry", "banana", "apple"]
```

### Merging Arrays

Use `concat()` or the spread operator to combine two arrays into a new one.

```js
let europeanCities = ["Paris", "Rome"];
let asianCities    = ["Tokyo", "Bangkok"];

let worldCities = europeanCities.concat(asianCities);
// or equivalently:
let worldCities = [...europeanCities, ...asianCities];
// → ["Paris", "Rome", "Tokyo", "Bangkok"]
```

### Quick method reference

```
  Method          Mutates?   Returns              Use when...
  ─────────────   ────────   ──────────────────   ──────────────────────────────────
  push(x)         ✅ yes     new length           adding to the end
  pop()           ✅ yes     removed element      removing from the end
  unshift(x)      ✅ yes     new length           adding to the front
  shift()         ✅ yes     removed element      removing from the front
  splice(i,n,x)   ✅ yes     removed elements     inserting/removing anywhere
  sort(fn)        ✅ yes     sorted array         reordering in place
  reverse()       ✅ yes     reversed array       flipping order in place

  slice(s,e)      ❌ no      new array            extracting a portion
  concat(arr)     ❌ no      new merged array     combining arrays
  map(fn)         ❌ no      new transformed arr  transforming every element
  filter(fn)      ❌ no      new filtered array   keeping elements that pass a test
  reduce(fn,init) ❌ no      single value         summing, grouping, aggregating
  find(fn)        ❌ no      element or undefined finding first match by condition
  findIndex(fn)   ❌ no      index or -1          finding position by condition
  indexOf(x)      ❌ no      index or -1          finding position by exact value
  includes(x)     ❌ no      true / false         checking if a value exists
  join(sep)       ❌ no      string               converting array to string
  some(fn)        ❌ no      true / false         checking if any element passes
  every(fn)       ❌ no      true / false         checking if all elements pass
```

---

## 8. Loops

**Source:** `part4/loopChallenges.js`, `part4/levelUpChallenges.js`

Loops let you run the same block of code multiple times without copy-pasting it. JavaScript has several kinds — each is suited to different situations.

### while — repeat as long as a condition is true

A `while` loop checks the condition **before** each iteration. If the condition is false right from the start, the body never runs. You're responsible for updating whatever the condition checks — otherwise you get an infinite loop.

```js
let sum = 0;
let i   = 1;

while (i <= 5) {
  sum += i;  // sum = sum + i
  i++;       // must move i forward or this loops forever
}
console.log(sum); // 15  (1+2+3+4+5)
```

### do...while — always run at least once

A `do...while` checks the condition **after** each iteration. This guarantees the body executes at least once, even if the condition is false. Useful when you need to run some setup code before knowing if you should keep going.

```js
let total = 0;
let k     = 1;

do {
  total += k;
  k++;
} while (k <= 3);
// total = 6  (1+2+3)
// The body ran once with k=1, checked k<=3 which was true, ran again, etc.
```

### for — when you know how many iterations you need

The classic `for` loop bundles the counter setup, condition, and increment all in one line. It's the most common loop for working with arrays by index.

```js
let numbers         = [2, 4, 6];
let multipliedNumbers = [];

for (let l = 0; l < numbers.length; l++) {
  multipliedNumbers.push(numbers[l] * 2);
}
// → [4, 8, 12]

// Step by step:
// l=0: numbers[0] * 2 = 4,  push 4
// l=1: numbers[1] * 2 = 8,  push 8
// l=2: numbers[2] * 2 = 12, push 12
// l=3: 3 < 3 is false → stop
```

### for...of — iterate over values in an array or string

`for...of` is cleaner than a regular `for` loop when you don't need the index — you just want each value directly. It works with any *iterable* (arrays, strings, Maps, Sets).

```js
let numbers     = [1, 2, 3, 4, 5];
let smallNumbers = [];

for (const num of numbers) {
  if (num === 4) break;     // stop the loop entirely
  smallNumbers.push(num);
}
// → [1, 2, 3]
```

### for...in — iterate over keys in an object

`for...in` gives you each property name (key) of an object one at a time. Use this when you want to walk through an object's keys — don't use it on arrays (it iterates numeric keys as strings, which is confusing).

```js
let citiesPopulation = {
  London:    8900000,
  "New York": 8400000,
  Berlin:    3500000,
  Paris:     2200000,
};

let cityNewPopulations = {};

for (const city in citiesPopulation) {
  if (city === "Berlin") break;  // stop when we reach Berlin
  cityNewPopulations[city] = citiesPopulation[city];
}
// → { London: 8900000, "New York": 8400000 }
// (Berlin triggers the break so it's not included)
```

### forEach — array method for iterating

`forEach` is a method on arrays that accepts a callback function and calls it once for each item. It's clean but has one important limitation: you cannot use `break` or `continue` inside it. Using `return` inside the callback skips that iteration (like `continue`), but does not exit the loop.

```js
let teaCollection = ["earl grey", "green tea", "chai", "oolong tea"];
let availableTeas = [];

teaCollection.forEach(function (tea) {
  if (tea === "chai") return;  // return = skip this item, NOT break
  availableTeas.push(tea);
});
// → ["earl grey", "green tea", "oolong tea"]  — chai was skipped
// Notice: "oolong tea" is still included because return didn't stop the loop
```

### break & continue

These two keywords give you precise control over loop flow.

- `break` — exits the entire loop immediately
- `continue` — skips the rest of the current iteration and jumps to the next one

```js
// continue — skip "Paris", process everything else
let cities      = ["London", "New York", "Paris", "Berlin"];
let visitedCities = [];

for (let i = 0; i < cities.length; i++) {
  if (cities[i] === "Paris") continue;  // skip Paris, move to Berlin
  visitedCities.push(cities[i]);
}
// → ["London", "New York", "Berlin"]
```

### Which loop should you use?

```
  Need the index while looping?    → for
  Just need each value in array?   → for...of  or  forEach
  Looping over object properties?  → for...in
  Don't know how many iterations?  → while
  Must run the body at least once?  → do...while
  Need to break out early?         → for / for...of (not forEach — no break)
```

---

## 9. Functions

**Source:** `part5/functions.js`

A function is a named block of code you can define once and run many times with different inputs. Functions are the primary tool for organising and reusing code. In JavaScript, functions are also *values* — you can store them in variables, pass them to other functions, and return them from functions. This is what makes JavaScript's style of programming so flexible.

### Function Declaration

The most basic form. You write the function once, then call it by name whenever you need it. The `return` keyword sends a value back to whoever called the function.

```js
function makeTea(typeOfTea) {
  return `Making ${typeOfTea}`;
  // anything after return is unreachable — the function exits at return
  console.log("this never runs");
}

let teaOrder = makeTea("lemon tea");
// teaOrder = "Making lemon tea"
```

### Arrow Function

A shorter syntax for writing functions, introduced in ES6. When the function body is a single expression, you can omit the `return` keyword and the curly braces — the value is returned implicitly.

```js
// Regular function
function calculateTotal(price, quantity) {
  return price * quantity;
}

// Arrow function — same thing, much shorter
const calculateTotal = (price, quantity) => price * quantity;

let totalCost = calculateTotal(499, 100); // 49900
```

Arrow functions also differ from regular functions in how they handle `this` — we'll cover that in section 15. For simple calculations or callbacks, arrow functions are the modern standard.

### Nested Functions

You can define a function inside another function. The inner function has access to the outer function's variables. This is useful for helper logic that's only relevant inside one specific function.

```js
function orderTea(teaType) {
  // confirmOrder is private to orderTea — can't call it from outside
  function confirmOrder() {
    return `Order confirmed for chai`;
  }
  return confirmOrder();  // call the inner function and return its result
}

let orderConfirmation = orderTea("chai");
// → "Order confirmed for chai"
```

### Higher-Order Functions

A **higher-order function** is any function that either receives a function as an argument, or returns a function. This might sound abstract, but you've already seen this with `forEach` — `forEach` is a higher-order function that takes your callback as an argument.

```js
function makeTea(typeOfTea) {
  return `Making ${typeOfTea}`;
}

// processTeaOrder receives a function (teaFunction) as an argument
function processTeaOrder(teaFunction) {
  return teaFunction("earl grey");  // calls whatever function was passed in
}

let order = processTeaOrder(makeTea);
// → "Making earl grey"
// makeTea is passed without (), because () would call it immediately
// We're passing the function itself as a value to be called later
```

This pattern is extremely common in JavaScript — event listeners, array methods like `.map()`, `.filter()`, and `.reduce()` all rely on it.

### Functions Returning Functions

A function can also return another function. The returned function "remembers" the variables from the outer function — this is the foundation of closures (covered in section 14).

```js
function createTeaMaker(name) {
  let score = 100;          // score lives in the outer function
  return function (teaType) {
    return `Making ${teaType} for ${name}, score: ${score}`;
    // name and score are remembered from the outer scope
  };
}

let teaMaker = createTeaMaker("hitesh");
// At this point, createTeaMaker has finished running...
// but the inner function still has access to name="hitesh" and score=100

let result = teaMaker("green tea");
// → "Making green tea for hitesh, score: 100"
```

### How the call stack works

When a function is called, JavaScript pushes it onto the call stack. When it returns, it's popped off. This is how JavaScript tracks "where am I in execution right now."

```
Call stack during teaMaker("green tea"):

  ┌───────────────────────────┐
  │  inner anonymous fn(...)  │  ← currently running
  ├───────────────────────────┤
  │  global / module scope    │  ← where teaMaker() was called
  └───────────────────────────┘
```

If you call too many nested functions without returning, you get a "Maximum call stack size exceeded" (stack overflow) error.

---

## 10. Object-Oriented Programming

**Source:** `part6/oop-master.js`, `part6/contructorFunction.js`

Object-Oriented Programming (OOP) is a way of organising code around *things* (objects) rather than just procedures. Instead of having separate variables and functions floating around, you bundle related data and behaviour together into one object.

### Object Literal

The simplest way to create an object is with curly braces. You define properties (data) and methods (functions) together.

```js
let car = {
  make:  "Toyota",
  model: "Camry",
  year:  2020,
  start() {              // method: a function attached to an object
    return `${this.make} car got started in ${this.year}`;
    // `this` refers to the car object itself
  },
};

car.start(); // → "Toyota car got started in 2020"
```

The `this` keyword inside an object method refers to the object that owns the method. When you call `car.start()`, `this` is `car`.

### Constructor Function

An object literal works for a single object. But if you need to create many similar objects (like many users, or many cars), you use a **constructor function**. By convention, constructor functions start with a capital letter to signal they should be called with `new`.

```js
function Person(name, age) {
  this.name = name;
  this.age  = age;
}

let john = new Person("John Doe", 20);
// john = { name: "John Doe", age: 20 }
```

The `new` keyword does four things automatically:
1. Creates a fresh empty object `{}`
2. Sets `this` inside the function to point to that empty object
3. Runs the constructor body (which adds properties via `this.x = ...`)
4. Returns the object (no `return` statement needed)

If you forget `new` and just call `Person(...)`, the function runs but `this` becomes the global object (or throws an error in strict mode). You can guard against this:

```js
function Drink(name) {
  if (!new.target) {
    throw new Error("Drink must be called with the new keyword");
  }
  this.name = name;
}

let tea    = new Drink("tea");      // ✅ works
let coffee = Drink("coffee");       // ❌ throws Error
```

### Classes — the modern syntax

ES6 introduced the `class` keyword. Under the hood, classes are just constructor functions with cleaner syntax — they don't introduce a new object model, they just make it easier to write. The `constructor` method runs when you call `new ClassName(...)`.

```js
class Vehicle {
  constructor(make, model) {
    this.make  = make;
    this.model = model;
  }

  start() {
    return `${this.model} is a car from ${this.make}`;
  }
}

let vehOne = new Vehicle("Toyota", "Corolla");
vehOne.start(); // → "Corolla is a car from Toyota"
```

### Inheritance — extends

Inheritance lets one class reuse the functionality of another. The child class gets all the parent's methods automatically, plus it can add its own.

```js
class Car extends Vehicle {
  drive() {
    return `${this.make}: This is an inheritance example`;
  }
}

let myCar = new Car("Toyota", "Corolla");
myCar.start();  // ✅ inherited from Vehicle — works without redeclaring it
myCar.drive();  // ✅ Car's own method
```

When a child class has a constructor, it must call `super()` first — this calls the parent's constructor to set up the shared properties before adding the child's own.

```
Inheritance chain — how JS finds methods:

  myCar.start() called
       │
       ▼  Check myCar object directly — not found
       │
       ▼  Check Car.prototype — not found (only has drive)
       │
       ▼  Check Vehicle.prototype — found start()! ✅
       │
       ▼  Object.prototype (toString, etc.)
       │
       ▼  null — end of chain, throw "not a function" if still not found
```

### The Four Pillars of OOP

#### Encapsulation — bundle data and restrict direct access

Encapsulation means keeping the internal state of an object private and only exposing controlled ways to interact with it. In JavaScript, private fields use the `#` prefix.

```js
class BankAccount {
  #balance = 0;    // private — cannot be accessed from outside the class

  deposit(amount) {
    this.#balance += amount;  // only the class itself can change #balance
    return this.#balance;
  }

  getBalance() {
    return `$ ${this.#balance}`;
  }
}

let account = new BankAccount();
account.deposit(500);
account.getBalance();      // ✅ → "$ 500"
console.log(account.#balance); // ❌ SyntaxError — private field
```

Why encapsulate? Because you don't want external code to set `account.#balance = 1000000` directly and bypass any validation. The `deposit` method can enforce rules (e.g. no negative deposits) before touching the balance.

#### Abstraction — hide complexity, expose simplicity

Abstraction means your object shows a simple interface to the outside world and hides all the messy implementation details inside. The user of an object shouldn't need to know *how* it works, only *what* it does.

```js
class CoffeeMachine {
  start() {
    // imagine: connect to DB, filter values, check pressure...
    return `Starting the machine...`;
  }

  brewCoffee() {
    // imagine: complex temperature calculations, timing...
    return `Brewing coffee`;
  }

  pressStartButton() {          // ← this is the public interface
    let msgOne = this.start();
    let msgTwo = this.brewCoffee();
    return `${msgOne} + ${msgTwo}`;
  }
}

let myMachine = new CoffeeMachine();
myMachine.pressStartButton();  // user only needs to know about this one method
```

#### Polymorphism — same method name, different behaviour

Polymorphism means a child class can override a method from its parent and provide its own implementation. The method has the same name but different logic depending on the object type.

```js
class Bird {
  fly() { return `Flying....`; }
}

class Penguin extends Bird {
  fly() { return `Penguins can't fly`; }  // override the parent's fly()
}

let bird    = new Bird();
let penguin = new Penguin();

bird.fly();     // → "Flying...."      ← uses Bird's implementation
penguin.fly();  // → "Penguins can't fly" ← uses Penguin's override
```

This is powerful when you have a list of mixed objects. You can call the same method on all of them and each object does the right thing for its type.

### Static Methods

A static method belongs to the **class itself**, not to any instance. You call it directly on the class. Useful for utility functions that don't need access to instance data.

```js
class Calculator {
  static add(a, b) {
    return a + b;
  }
}

Calculator.add(2, 3);            // ✅ 5 — called on the class
// new Calculator().add(2, 3);  // ❌ TypeError — not available on instances
```

Think of `Math.random()` and `Math.floor()` — you never do `new Math()`. Those are static.

### Getters & Setters

Getters and setters let you define properties that *look* like regular properties but actually run code when accessed or assigned. This lets you add validation or computed logic while keeping the simple `emp.salary` syntax.

```js
class Employee {
  #salary;
  constructor(name, salary) {
    if (salary < 0) throw new Error("Salary cannot be negative");
    this.name    = name;
    this.#salary = salary;
  }

  get salary() {
    return `You are not allowed to see salary`;
    // The actual #salary is hidden; the getter returns a message instead
  }

  set salary(value) {
    if (value < 0) {
      console.error("Invalid Salary");
    } else {
      this.#salary = value;  // validation before setting
    }
  }
}

let emp      = new Employee("Alice", 50000);
emp.salary;          // looks like property access, but calls the getter
emp.salary = 60000;  // looks like assignment, but calls the setter
```

---

## 11. Prototypes

**Source:** `part6/prototypes.js`, `part9_adv/prototypalInheritance.js`

Before classes existed in JavaScript, *prototypes* were the only way to share methods between objects. Understanding prototypes helps you understand what's actually happening under the hood when you use classes — because classes are built on prototypes.

Every JavaScript object has a hidden internal link to another object called its **prototype**. When you try to access a property that doesn't exist directly on an object, JavaScript automatically walks up this chain of prototypes looking for it. This is called the **prototype chain**.

```js
// __proto__ sets the prototype directly (old style — just for learning)
let computer = { cpu: 12 };
let lenovo   = {
  screen: "HD",
  __proto__: computer,   // lenovo's prototype is computer
};

lenovo.cpu;   // → 12
// lenovo doesn't have a "cpu" property, so JS looks at computer → finds it
```

The modern way to get/set prototypes:

```js
let genericCar = { tyres: 4 };
let tesla      = { driver: "AI" };

Object.setPrototypeOf(tesla, genericCar);     // tesla inherits from genericCar
Object.getPrototypeOf(tesla);                 // → { tyres: 4 }
tesla.tyres;                                  // → 4 (found via prototype chain)
```

### Adding methods via Prototype

Instead of defining a method inside each object individually (which wastes memory), you define it once on the constructor's prototype and all instances share it.

```js
function Animal(type) {
  this.type = type;
  // NOT adding speak() here — that would create a new function for each instance
}

// Add speak() once on the shared prototype
Animal.prototype.speak = function () {
  return `${this.type} makes a sound`;
};

let dog = new Animal("Dog");
let cat = new Animal("cat");

dog.speak(); // → "Dog makes a sound"
cat.speak(); // → "cat makes a sound"
// Both dog and cat share the same speak function — no duplication
```

You can even extend built-in prototypes (though you should avoid this in production code — it affects all arrays everywhere):

```js
Array.prototype.hitesh = function () {
  return `Custom method on ${this}`;
};

[1, 2, 3].hitesh();       // works
[1, 2, 3, 4, 5, 6].hitesh(); // works on any array
```

### Prototypal Inheritance

Constructor functions can inherit from each other through the prototype chain. This is exactly what `class` + `extends` does internally.

```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log(`Hello, my name is ${this.name}`);
};

let hitesh = new Person("hitesh");
hitesh.greet(); // → "Hello, my name is hitesh"
```

```
How hitesh.greet() is resolved:

  hitesh.greet() called
       │
       ▼  hitesh = { name: "hitesh" }  — does it have greet? No.
       │
       ▼  Person.prototype = { greet: ƒ }  — found greet! ✅ Call it.
       │
       ▼  Object.prototype = { toString, valueOf, ... }
       │
       ▼  null — chain ends here, ReferenceError if still not found
```

When you write a `class` in JavaScript, the engine creates this exact prototype structure for you automatically. Classes are syntactic sugar — prototypes are the actual mechanism.

---

## 12. DOM Manipulation & Events

**Source:** `part7/script.js`, `part8/script.js`

When a browser loads an HTML page, it parses it and builds a tree structure called the **Document Object Model (DOM)**. Every HTML tag becomes a node in this tree. JavaScript can read and modify this tree in real time — that's how webpages change dynamically without reloading.

```
Browser's DOM tree for a typical page:

  document
     │
     └── <html>
           ├── <head>
           │     └── <title>My Page</title>
           └── <body>
                 ├── <button id="changeTextButton">Change Text</button>
                 ├── <p id="myParagraph">Original text</p>
                 └── <ul id="citiesList">
                       ├── <li class="teaItem">Green Tea</li>
                       └── <li class="teaItem">Chai</li>
```

### Selecting Elements

Before you can change something, you need to grab a reference to it. `getElementById` is fast and returns exactly one element. `querySelector` is more flexible — it accepts any CSS selector.

```js
let paragraph    = document.getElementById("myParagraph");
let firstTeaItem = document.querySelector(".teaItem");       // first match only
let allTeaItems  = document.querySelectorAll(".teaItem");    // NodeList of all matches
```

### Modifying Content & Style

Once you have an element, you can read and update its text or styles directly.

```js
// Change text content
paragraph.textContent = "The paragraph has been changed";

// Change inline styles — property names are camelCase in JS (not CSS's kebab-case)
let coffeeType = document.getElementById("coffeeType");
coffeeType.textContent          = "Espresso";
coffeeType.style.backgroundColor = "brown";  // CSS: background-color
coffeeType.style.padding         = "5px";
```

### Modifying Classes

Rather than setting inline styles directly, it's better practice to toggle CSS classes. Your CSS file defines the `.highlight` class, and JS just adds/removes it.

```js
element.classList.add("highlight");       // adds the class
element.classList.remove("highlight");    // removes it
element.classList.toggle("highlight");    // adds it if absent, removes if present
// toggle is perfect for on/off switches like dark mode
```

### Creating & Removing Elements

You're not limited to changing existing elements — you can create brand new ones and insert them, or remove existing ones.

```js
// Create a new <li>, set its text, and add it to the list
let newItem         = document.createElement("li");
newItem.textContent = "Eggs";
document.getElementById("shoppingList").appendChild(newItem);
// Now "Eggs" appears at the bottom of the shopping list

// Remove the last task in a list
let taskList = document.getElementById("taskList");
taskList.lastElementChild.remove();

// Add a class to the first child
let citiesList = document.getElementById("citiesList");
citiesList.firstElementChild.classList.add("highlight");
```

### Events

Events are things that happen — a user clicks a button, submits a form, the page finishes loading. You "listen" for these events with `addEventListener` and provide a callback function that runs when the event fires.

```js
// Click event
document.getElementById("changeTextButton").addEventListener("click", function () {
  let paragraph = document.getElementById("myParagraph");
  paragraph.textContent = "the paragraph is changed";
});

// Double-click event
document.getElementById("clickMeButton").addEventListener("dblclick", function () {
  alert("chaicode");
});

// Form submit — event.preventDefault() stops the page from refreshing
document.getElementById("feedbackForm").addEventListener("submit", function (event) {
  event.preventDefault();  // without this, the page would reload on submit
  let feedback = document.getElementById("feedbackInput").value;
  document.getElementById("feedbackDisplay").textContent = `Feedback: ${feedback}`;
});

// DOMContentLoaded — runs when the HTML is fully parsed (before images load)
document.addEventListener("DOMContentLoaded", function () {
  document.getElementById("domStatus").textContent = "DOM fully loaded";
});
```

### Event Delegation

Imagine a list with 1000 items. Adding a separate click listener to each `<li>` would be slow and memory-heavy. Instead, you add ONE listener to the parent element. When a child is clicked, the event *bubbles up* to the parent, and you check `event.target` to see what was actually clicked.

```js
document.getElementById("teaList").addEventListener("click", function (event) {
  // event.target is the exact element that was clicked
  if (event.target && event.target.matches(".teaItem")) {
    alert("You selected: " + event.target.textContent);
  }
});
```

This works because of **event bubbling** — when an element fires an event, the event also fires on every parent element up to the document root.

```
Event Bubbling visualised:

  User clicks <li class="teaItem">Green Tea</li>
       │
       ▼  Event fires on <li>
       │  (bubbles up automatically)
       ▼  Event fires on <ul id="teaList">  ← our listener catches it here
       │  event.target is still the <li> that was originally clicked
       ▼  Event fires on <body>
       ▼  Event fires on <html>
       ▼  Event fires on document
```

### BOM — Browser Object Model

The **DOM** represents the *page content* (the HTML tree). The **BOM** is everything else the browser exposes about itself — the window, the screen, the URL, the browser info. The global `window` object is the top of the BOM, and it's so central that you rarely type `window.` explicitly — `document`, `alert()`, `setTimeout()` are all actually `window.document`, `window.alert()`, etc.

```
Where BOM fits relative to the DOM:

  window  (the BOM root — represents the browser tab itself)
     │
     ├── document      ← the DOM (the page's HTML tree — section 12 above)
     ├── navigator      ← info about the browser/device
     ├── location       ← info about the current URL
     ├── screen         ← info about the user's physical display
     └── history        ← back/forward navigation
```

**`navigator`** tells you about the browser and device running the page. Useful for feature detection or showing device-specific behaviour.

```js
console.log(navigator.userAgent);  // string identifying the browser, e.g. "Mozilla/5.0 ..."
console.log(navigator.language);   // "en-US" — user's preferred language
console.log(navigator.onLine);     // true/false — is the device connected to the internet
```

**`location`** represents the current page's URL. You can read parts of it, or assign to it to navigate the browser to a new page (this reloads the page, like clicking a link).

```js
console.log(location.href);      // full URL, e.g. "https://chaicode.com/learn?tab=js"
console.log(location.hostname);  // "chaicode.com"
console.log(location.pathname);  // "/learn"
console.log(location.search);    // "?tab=js" — the query string

location.href = "https://chaicode.com"; // navigates the browser to a new page
location.reload();                       // reloads the current page
```

**`screen`** describes the user's physical display — not the browser window, the actual monitor. Useful for adapting layouts to very large or very small screens.

```js
console.log(screen.width);   // full screen width in pixels, e.g. 1920
console.log(screen.height);  // full screen height in pixels, e.g. 1080
```

> Quick distinction: `screen` = the physical monitor. `window.innerWidth` / `window.innerHeight` = the visible browser viewport, which is usually smaller than the screen.

The DOM lets you change *what's on the page*; the BOM lets you interact with *the browser around the page*.

---

## 13. The Event Loop

**Source:** `part9_adv/explore1.js`

JavaScript is **single-threaded** — it can only do one thing at a time. There is a single call stack, and JavaScript works through it top to bottom. So how does it handle things like timers, network requests, and user events without freezing the page? The answer is the **event loop**.

When JavaScript encounters something that takes time (like `setTimeout`, `fetch`, or an event listener), it hands it off to the browser's Web APIs. The browser handles the waiting in the background. When the time is up or the data arrives, the callback goes into a queue. The event loop's job is to check: "Is the call stack empty? If yes, move the next callback from the queue into the stack."

```js
function sayHello() {
  console.log("I would like to say Hello");
}

setTimeout(() => {
  sayHello();         // this goes to Web API, waits 4 seconds
}, 4000);

console.log("chaicode");   // this runs immediately

for (let index = 0; index < 10; index++) {
  console.log(index);      // 0-9 all run immediately
}

// Output order:
// "chaicode"
// 0
// 1
// 2  ... up to 9
// (4 seconds pass...)
// "I would like to say Hello"
```

Even though `setTimeout` appears first in the code, `console.log("chaicode")` and the entire `for` loop run first. The `setTimeout` callback only runs after the call stack is empty.

```
Event Loop Architecture:

  JS Engine (single thread)
  ┌──────────────────────┐      Browser Web APIs
  │      Call Stack      │      ┌────────────────────────┐
  │  ┌────────────────┐  │      │  setTimeout(cb, 4000)  │
  │  │  for loop body │  │ ──►  │  fetch() / XHR         │
  │  └────────────────┘  │      │  DOM event listeners   │
  │  ┌────────────────┐  │      └────────────┬───────────┘
  │  │  console.log() │  │                   │  (timer fires / request done)
  │  └────────────────┘  │                   ▼
  └──────────┬───────────┘      ┌────────────────────────┐
             │                  │   Callback Queue        │
             │                  │   [ sayHello callback ] │
             │                  └────────────┬───────────┘
             │                               │
             └──── Event Loop watches ───────┘
                   "Call stack empty?
                    Push next callback in."
```

**Key insight:** `setTimeout(fn, 0)` does NOT mean "run this immediately." It means "run this as soon as the call stack is empty." Even with a delay of 0, all synchronous code runs first.

---

## 14. Closures

**Source:** `part9_adv/clousure.js`, `part5/functions.js`

A **closure** is created when a function "remembers" variables from its outer scope, even after the outer function has finished executing. This is not a special thing you set up — it happens automatically in JavaScript whenever a function is defined inside another function.

The classic example: `outer()` runs and returns the inner function. Normally, once a function finishes, its local variables are garbage collected (cleaned up from memory). But because the inner function *uses* `counter`, JavaScript keeps `counter` alive as long as `increment` exists.

```js
function outer() {
  let counter = 4;       // this variable would normally be cleaned up after outer() returns
  return function () {   // but this inner function uses counter, so it stays alive
    counter++;
    return counter;
  };
}

let increment = outer();  // outer() has finished, but counter lives on in the closure
console.log(increment()); // 5
console.log(increment()); // 6
console.log(increment()); // 7
// Each call continues from where the last left off — counter persists
```

```
Closure memory model:

  outer() finishes and returns the inner function
  The inner function carries a "backpack" of the variables it referenced:

  increment  ────►  [ anonymous function ]
                          │
                          │  has access to closure scope:
                          ▼
                    ┌───────────────────────────┐
                    │  counter = 4 → 5 → 6 → 7  │  ← persists in memory
                    └───────────────────────────┘

  counter is NOT garbage collected as long as `increment` holds the function
```

### Why closures matter — data privacy

Closures let you create variables that are private to a function but persistent across calls. There's no other way to achieve this in plain JavaScript (without classes).

```js
function createTeaMaker(name) {
  let score = 100;              // private — no external code can touch this
  return function (teaType) {
    return `Making ${teaType} for ${name}, score: ${score}`;
  };
}

let teaMaker = createTeaMaker("hitesh");
teaMaker("green tea");   // → "Making green tea for hitesh, score: 100"
teaMaker("chai");        // → "Making chai for hitesh, score: 100"

// There's no way to read or change `score` from outside — it's trapped in the closure
```

This is the same concept used by JavaScript module patterns, React's `useState` hook internally, and countless libraries. Once you understand closures, a lot of "magic" in JavaScript becomes clear.

---

## 15. `this` Context, bind / call / apply

**Source:** `part9_adv/thisContext.js`

`this` is a special keyword in JavaScript that refers to the object the current function is running as a method of. Unlike most languages, `this` in JavaScript is not fixed at definition time — it's determined at *call time*, based on how the function is invoked. This is what makes it confusing.

```js
const person = {
  name: "Hitesh",
  greet() {
    console.log(`Hi, I am ${this.name}`);
  },
};

person.greet();          // → "Hi, I am Hitesh"
// this = person, because greet was called ON person (person.greet)
```

Now here's where it breaks:

```js
const greetFunction = person.greet;  // store the function in a variable
greetFunction();                      // → "Hi, I am undefined"
// this is no longer person — it's the global object (or undefined in strict mode)
// because greetFunction() is called without any object before the dot
```

The function is the same function. But *how it was called* changed `this`. This is the core of `this` confusion: it's about the call site, not where the function was defined.

### bind — permanently attach a `this` value

`bind` creates a **new function** with `this` locked to whatever you pass in. It doesn't call the function immediately — it just returns a new version with a fixed context.

```js
const boundGreet = person.greet.bind({ name: "John" });
boundGreet();  // → "Hi, I am John"
// this is now permanently { name: "John" } regardless of how it's called
```

### call and apply — call a function with a specific `this` right now

Unlike `bind`, `call` and `apply` invoke the function immediately with a specified `this`.

```js
function introduce(city, lang) {
  return `${this.name} from ${city} speaks ${lang}`;
}

// call: arguments passed individually, comma-separated
introduce.call({ name: "Hitesh" }, "Jaipur", "Hindi");
// → "Hitesh from Jaipur speaks Hindi"

// apply: arguments passed as an array
introduce.apply({ name: "Hitesh" }, ["Jaipur", "Hindi"]);
// → same result
```

```
bind / call / apply — when to use which:

  bind()    →  You need a reusable function with a fixed this, called later
  call()    →  You want to call it NOW with individual arguments
  apply()   →  You want to call it NOW but your args are already in an array
```

### Arrow functions and this

Arrow functions do **not** have their own `this`. They inherit `this` from the surrounding code where they were *defined*. This makes them predictable for callbacks and avoids the "lost this" problem.

```js
const person = {
  name: "Hitesh",
  greet: () => {
    console.log(`Hi, I am ${this.name}`);
    // 'this' here is NOT person — it's whatever 'this' was in the outer scope
  },
};
person.greet(); // → "Hi, I am undefined"
// Arrow functions should NOT be used as object methods for this reason
```

Arrow functions are ideal as callback functions inside methods (e.g. `setTimeout(() => { ... })`) because they preserve the outer `this`.

---

## 16. Promises

**Source:** `part9_adv/promises.js`

A **Promise** is an object that represents the result of an asynchronous operation that hasn't necessarily completed yet. Instead of blocking JavaScript while waiting for a network response or a timer, a Promise lets you say: "when this is done, run this code."

A Promise is always in one of three states:
- **Pending** — the operation hasn't completed yet
- **Fulfilled** — it completed successfully (via `resolve()`)
- **Rejected** — it failed (via `reject()`)

Once a Promise settles (either fulfilled or rejected), it never changes state.

```
Promise state machine:

  ┌──────────────────┐
  │    pending       │──── resolve(value) ────► fulfilled ──► .then(value => ...)
  │  (in progress)   │
  │                  │──── reject(reason) ────► rejected  ──► .catch(reason => ...)
  └──────────────────┘
```

```js
function fetchData() {
  return new Promise((resolve, reject) => {
    // The Promise constructor receives a function with two callbacks:
    // - resolve: call this when the operation succeeds
    // - reject: call this when it fails

    setTimeout(() => {
      let success = true;
      if (success) {
        resolve("Data fetched successfully");  // passes value to .then()
      } else {
        reject("Error fetching data");         // passes reason to .catch()
      }
    }, 3000);  // simulates a 3 second network request
  });
}

fetchData()
  .then((data) => {
    console.log(data);           // → "Data fetched successfully"
    return data.toLowerCase();   // returning from .then() passes the value to the next .then()
  })
  .then((value) => {
    console.log(value);          // → "data fetched successfully"
  })
  .catch((error) => {
    console.error(error);        // catches any rejection from fetchData() OR from any .then() above
  });
```

### Promise Chaining

Each `.then()` returns a new Promise. If you return a value inside `.then()`, it wraps it in a resolved Promise automatically, which the next `.then()` receives. This lets you chain async operations in a readable pipeline instead of nesting callbacks inside callbacks (the old "callback hell" pattern).

```
Chained Promises:

  fetchData()         — returns a Promise
       │
    .then(data => ...) — runs when resolved, returns a new Promise
       │
    .then(value => ...) — runs when that resolves
       │
    .catch(error => ...) — catches any rejection anywhere above in the chain
```

---

## 17. Async / Await

**Source:** `part9_adv/asyncAwait.js`, `part9_adv/asyncAwaitpart2.js`

`async/await` is built on top of Promises but gives you a way to write asynchronous code that *looks* and *reads* like synchronous code. Instead of chaining `.then()`, you just write `await` in front of a Promise and JavaScript pauses that function until the Promise resolves. Errors are caught with a regular `try/catch` block.

The `async` keyword marks a function as asynchronous — this means it always returns a Promise, and you're allowed to use `await` inside it.

```js
async function getUserData() {
  try {
    console.log("Fetching user data...");

    // await pauses getUserData() here until fetchUserData() resolves or rejects
    const userData = await fetchUserData();

    console.log("User data fetched successfully");
    console.log("User data:", userData);
  } catch (error) {
    // If fetchUserData() rejects, the error lands here — same as .catch()
    console.log("Error fetching data", error);
  }
}

getUserData();
// "Fetching user data..." prints immediately
// then the function pauses waiting for fetchUserData()
// if it resolves: prints user data
// if it rejects: catch block runs
```

The key thing: `await` does not block the browser or Node.js. It only "pauses" the current `async` function. Everything outside that function keeps running normally.

### Promise.all — fetch multiple things in parallel

If you `await` multiple Promises one after another, they run sequentially — you wait for the first to finish before starting the second. Often you want them to run at the same time. `Promise.all` starts all the Promises at once and waits until **all** of them have resolved.

```js
async function getBlogData() {
  try {
    console.log("Fetching blog data");

    // Sequential — total wait: 2s + 3s = 5s  (slow)
    // const postData    = await fetchPostData();
    // const commentData = await fetchCommentData();

    // Parallel — total wait: max(2s, 3s) = 3s  (fast)
    const [postData, commentData] = await Promise.all([
      fetchPostData(),     // starts immediately
      fetchCommentData(),  // also starts immediately — doesn't wait for postData
    ]);

    console.log(postData);    // → "Post Data fetched"
    console.log(commentData); // → "Comment data fetched."
    console.log("fetch complete");
  } catch (error) {
    // If ANY promise in the array rejects, the whole Promise.all rejects
    console.error("Error fetching blog data", error);
  }
}
```

```
Sequential vs Parallel:

  Sequential (slow — each waits for previous):
  ─────────────────────────────────────────────
  Time: 0s ──[fetchPost 2s]──► 2s ──[fetchComment 3s]──► 5s DONE

  Parallel with Promise.all (fast — both run simultaneously):
  ───────────────────────────────────────────────────────────
  Time: 0s ──[fetchPost 2s]──────────────────────► 2s ✓
            ──[fetchComment 3s]────────────────────────────► 3s ✓
                                                              ↑
                                                    3s DONE (max of the two)
```

Use `Promise.all` whenever the operations are independent of each other — loading user data, posts, and comments simultaneously instead of one at a time.

---

## 18. Generators & Iterators

**Source:** `part9_adv/generatorIterator.js`

A **generator** is a special type of function that can pause its execution and resume from where it left off. Regular functions run from start to finish in one shot. A generator can yield values one at a time, pausing after each one.

Generators are defined with a `*` after the `function` keyword. Inside, you use `yield` instead of `return`. When you call `gen.next()`, the function runs until the next `yield`, pauses there, and returns an object `{ value: ..., done: false }`. When there are no more `yield` statements, `done` becomes `true`.

```js
function* numberGenerator() {
  console.log("About to yield 1");
  yield 1;   // pause here, return 1
  console.log("About to yield 2");
  yield 2;   // pause here, return 2
  console.log("About to yield 3");
  yield 3;   // pause here, return 3
  // function ends after this — done: true
}

let gen    = numberGenerator();
let genTwo = numberGenerator(); // completely independent — has its own state

console.log(gen.next().value);    // "About to yield 1" then → 1
console.log(gen.next().value);    // "About to yield 2" then → 2
console.log(gen.next().value);    // "About to yield 3" then → 3
console.log(gen.next().value);    // undefined (generator is done)

console.log(genTwo.next().value); // 1  — genTwo starts fresh from the beginning
console.log(genTwo.next().value); // 2
```

```
Generator execution flow:

  gen = numberGenerator()     ← doesn't run the body yet

  gen.next() called:
       ├── runs until yield 1
       ├── pauses, returns { value: 1, done: false }
       └── resumes from here on next call

  gen.next() called again:
       ├── runs from after yield 1 until yield 2
       ├── pauses, returns { value: 2, done: false }

  gen.next() called again:
       ├── runs from after yield 2 until yield 3
       └── returns { value: 3, done: false }

  gen.next() called again:
       └── no more yields → returns { value: undefined, done: true }
```

Generators are useful for lazy sequences (generating values on demand), infinite sequences, and as the internal mechanism behind `async/await` in older implementations. They give you fine-grained control over iteration.

---

## 19. Modules (CJS & ESM)

**Source:** `part9_adv/mathOperationsC.js`, `part9_adv/appC.js`, `part9_adv/mathOperationsM.js`, `part9_adv/appM.js`

As projects grow, keeping all code in one file becomes unmanageable. Modules let you split your code across multiple files. Each file is its own module — it keeps its variables and functions private by default. You explicitly choose what to export (make available to other files) and what to import (use from other files).

JavaScript has two module systems: **CommonJS (CJS)**, the original Node.js system, and **ES Modules (ESM)**, the modern standard that works in both browsers and Node.js.

### CommonJS (CJS)

CJS was built for Node.js. Files are treated as modules by default. `module.exports` is the object that other files receive when they `require()` your file.

**Exporting (`mathOperationsC.js`)**

```js
function add(a, b)      { return a + b; }
function subtract(a, b) { return a - b; }
function multiply(a, b) { return a * b; }

// Attach whatever you want to export onto module.exports
// You can export as many named things as you like
module.exports = { add, subtract, multiply };
```

**Importing (`appC.js`)**

```js
// require() runs the file and returns whatever module.exports was set to
const mathOperations = require("./mathOperationsC.js");

console.log(mathOperations.add(2, 2));  // → 4
```

CJS `require()` is **synchronous** — the file is loaded and executed immediately at the point of the `require` call. This is fine in Node.js but doesn't work well in browsers where loading files asynchronously is essential.

### ES Modules (ESM)

ESM is the standard built into the JavaScript language specification. It uses `import` and `export` keywords. Files need a `.mjs` extension or `"type": "module"` in `package.json` for Node.js to treat them as ES modules.

ESM has two flavours of exports: **named exports** (you can have many) and a **default export** (only one per file). When importing, named imports use `{ }` and default imports don't.

**Exporting (`mathOperationsM.js`)**

```js
// Named exports — can have as many as you want
export function add(a, b)      { return a + b; }
export function subtract(a, b) { return a - b; }

// Default export — only one per file, no name required at export
export default function multiply(a, b) { return a * b; }
```

**Importing (`appM.js`)**

```js
// Default import — name it anything you want
import multiply from "./mathOperationsM.js";

// Named imports — must use the exact exported names, in { }
import { add, subtract } from "./mathOperationsM.js";

console.log(multiply(2, 2));  // → 4
console.log(add(2, 2));       // → 4
```

### CJS vs ESM — what's the difference?

```
  Feature              CJS (require/exports)      ESM (import/export)
  ───────────────────  ────────────────────────   ────────────────────────────
  Syntax               require() / module.exports  import / export
  Execution            synchronous (blocking)      asynchronous (non-blocking)
  Works in browsers?   ❌ (needs bundler)           ✅ natively
  Default in Node.js?  ✅ yes                       needs "type":"module"
  Tree-shaking         ❌ harder — whole file runs  ✅ bundlers can eliminate unused exports
  Top-level await      ❌ not supported              ✅ supported
  Dynamic imports      require() anywhere           import() — returns a Promise
```

In modern projects (React, Vue, Node.js APIs), ESM is the preferred choice. CJS still appears in older Node.js projects and libraries.

---

## Quick Reference — Mental Model

```
JavaScript Foundation — how everything connects:

  ┌───────────────────────────────────────────────────────────────────┐
  │  Foundations                                                       │
  │  Data Types → Variables (let/const) → Operators → Conditionals    │
  └──────────────────────────────┬────────────────────────────────────┘
                                 │
                  ┌──────────────┴──────────────┐
                  ▼                             ▼
  ┌───────────────────────┐      ┌──────────────────────────┐
  │  Data Structures      │      │  Logic & Control Flow    │
  │  Arrays (ordered)     │      │  Loops: for/while/       │
  │  Objects (key-value)  │      │  for-of/for-in/forEach   │
  │  Primitives vs Refs   │      │  break / continue        │
  └───────────┬───────────┘      └──────────────────────────┘
              │
              ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Functions                                                    │
  │  Declaration → Arrow → Nested → Higher-Order → Closures      │
  └──────────────────────────┬───────────────────────────────────┘
                             │
             ┌───────────────┴───────────────┐
             ▼                               ▼
  ┌──────────────────────┐     ┌─────────────────────────────┐
  │  OOP & Prototypes    │     │  DOM & Events (browser)     │
  │  Objects/Classes     │     │  getElementById             │
  │  Inheritance         │     │  addEventListener           │
  │  Encapsulation       │     │  Event Delegation           │
  │  Prototype chain     │     │  createElement / remove     │
  └──────────────────────┘     └─────────────────────────────┘
             │
             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Async JavaScript                                             │
  │  Event Loop → Callbacks → Promises → async/await            │
  │  this / bind / call / apply → Generators → Modules           │
  └──────────────────────────────────────────────────────────────┘
```

---

*All examples taken directly from the `jsFoundation/` course files.*
