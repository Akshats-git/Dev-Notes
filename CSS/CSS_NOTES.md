# CSS — Complete Guide

> Based on the code in `code+files/css/` — every concept explained with context, not just syntax. Written in plain English, following the projects in this folder from basics to Bootstrap.

---

## Table of Contents

1. [What Is CSS?](#1-what-is-css)
2. [Three Ways to Add CSS](#2-three-ways-to-add-css)
3. [Anatomy of a CSS Rule](#3-anatomy-of-a-css-rule)
4. [Selectors](#4-selectors)
5. [The Cascade, Specificity, and Inheritance](#5-the-cascade-specificity-and-inheritance)
6. [Colors, Units, and Values](#6-colors-units-and-values)
7. [The Box Model](#7-the-box-model)
8. [Display: Block, Inline, and Inline-Block](#8-display-block-inline-and-inline-block)
9. [Pseudo-classes and Pseudo-elements](#9-pseudo-classes-and-pseudo-elements)
10. [Positioning](#10-positioning)
11. [Flexbox](#11-flexbox)
12. [CSS Grid](#12-css-grid)
13. [Transitions and Transforms](#13-transitions-and-transforms)
14. [Backgrounds, Gradients, and Shadows](#14-backgrounds-gradients-and-shadows)
15. [Responsive Design and Media Queries](#15-responsive-design-and-media-queries)
16. [Project Walkthrough: Login Card](#16-project-walkthrough-login-card)
17. [Project Walkthrough: Navbars](#17-project-walkthrough-navbars)
18. [Bootstrap: A CSS Framework](#18-bootstrap-a-css-framework)
19. [Quick Reference — Mental Model](#19-quick-reference--mental-model)

---

## 1. What Is CSS?

CSS stands for **Cascading Style Sheets**. If HTML is the structure of a page (the walls and rooms), CSS is the styling (the paint, furniture, and layout). HTML says "this is a heading, this is a paragraph, this is a button". CSS says "make the heading orange, give the button rounded corners, put these boxes side by side".

Without CSS, every website would look like a plain black-and-white document: default fonts, no colors, no spacing, everything stacked in a single column. CSS is what makes pages look designed.

```
The division of labor:

  HTML  →  what the content IS      (heading, paragraph, image, form)
  CSS   →  how the content LOOKS    (color, size, spacing, position)
  JS    →  how the content BEHAVES  (clicks, animations, live updates)
```

The word "cascading" is important. Many style rules can apply to the same element at once, and CSS has a clear set of rules that decides which one wins. We cover that in section 5.

CSS works by **selecting** HTML elements and then **declaring** how they should look. You point at something ("all paragraphs") and describe its style ("blue text, 25 pixels tall").

---

## 2. Three Ways to Add CSS

**Source:** `01_basics/index.html`, `01_basics/style.css`

There are three places you can put CSS, and the very first project shows all three at once. Understanding the difference matters because they behave differently and one is strongly preferred.

### 1. Inline CSS (in a `style` attribute)

You attach styles directly to a single element using the `style` attribute. From `01_basics/index.html`:

```html
<h1 style="color: orangered">Lorem ipsum dolor sit amet.</h1>
```

This styles only this one heading. It is quick but messy: you cannot reuse it, and it mixes structure with styling. Avoid it for real work.

### 2. Internal CSS (in a `<style>` tag)

You write CSS inside a `<style>` block in the page's `<head>`. From the same file:

```html
<head>
    <style>
      .browntext {
        color: brown;
      }
    </style>
</head>
```

Now any element with `class="browntext"` on this page turns brown. This is fine for a single page, but the styles cannot be shared across multiple pages.

### 3. External CSS (a separate `.css` file)

You put your CSS in its own file and link it from the HTML. This is the professional standard. The project links `style.css`:

```html
<link rel="stylesheet" href="style.css" />
```

And `style.css` contains:

```css
p {
  color: cornflowerblue;
  font-size: 25px;
  padding: 200px;
}
```

One CSS file can style your entire website, and updating one file changes every page at once.

```
Which to use:

  Inline    → one element only. Avoid it (hard to maintain).
  Internal  → one page. OK for tiny demos.
  External  → whole site. THE recommended way. One file, many pages.

  Order of preference:  External  >  Internal  >  Inline
```

> **Notice a conflict in the basics project:** the `<p>` has `class="browntext"` (brown, from internal CSS) but `style.css` also says `p { color: cornflowerblue }`. Which wins? The class selector `.browntext` beats the type selector `p`, so the paragraph is brown. This is specificity, covered in section 5.

---

## 3. Anatomy of a CSS Rule

Every piece of CSS follows the same shape. Learn these words because the rest of the guide uses them constantly.

```
              declaration block
                     │
     selector    ┌───┴────────────────┐
        │        │                    │
        ▼        ▼                    ▼
        p  {  color: cornflowerblue;  font-size: 25px;  }
              └─┬─┘  └──────┬───────┘
             property     value
              └────── declaration ─────┘
```

- **Selector**: what you are targeting (here, all `<p>` elements).
- **Declaration block**: everything inside the curly braces `{ }`.
- **Declaration**: one `property: value;` pair.
- **Property**: the thing you are changing (`color`, `font-size`, `padding`).
- **Value**: what you are setting it to (`cornflowerblue`, `25px`).

Each declaration ends with a semicolon `;`. Forgetting a semicolon breaks the declarations after it, so always include it.

You can also add **comments** to explain your CSS. Anything between `/*` and `*/` is ignored by the browser. The selectors project uses these to label each example:

```css
/* 1. Universal Selector */
* {
  margin: 0;
}
```

---

## 4. Selectors

**Source:** `03_selectors/selectors.html`

Selectors are how you choose which elements to style. The selectors project is a full tour of the main types, so we will walk through all twelve it demonstrates. This is one of the most important sections in the whole guide.

### 1. Universal selector `*`

The `*` selects **every element** on the page. It is most often used to reset default spacing:

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
```

This wipes out the browser's built-in margins and paddings so you start from a clean slate. The `box-sizing: border-box` part is explained in the box model section.

### 2. Type (element) selector

Selects all elements of a given tag name:

```css
p {
  font-size: 16px;
  margin-bottom: 10px;
}
```

Every `<p>` on the page gets this style.

### 3. Class selector `.`

Selects all elements that have a given `class`. Written with a dot before the name:

```css
.highlight {
  background-color: #21a2c9;
}
```

```html
<p class="highlight">This paragraph is highlighted.</p>
```

Classes are the workhorse of CSS. You can reuse the same class on as many elements as you like, and an element can have several classes at once.

### 4. ID selector `#`

Selects the single element with a given `id`. Written with a hash before the name:

```css
#header {
  background-color: #860303;
}
```

```html
<header id="header">This is the header</header>
```

An `id` must be unique on a page (only one element can have it). Use classes for reusable styles and reserve ids for one-off elements.

### 5. Attribute selector `[ ]`

Selects elements based on an attribute and its value:

```css
input[type="text"] {
  border: 2px solid #21a2c9;
}
```

This targets only text inputs, leaving password or email inputs alone.

### 6. Descendant selector (space)

A space between two selectors means "any of the second that is **inside** the first, at any depth":

```css
article p {
  font-style: italic;
  color: #fff;
}
```

Every `<p>` anywhere inside an `<article>` gets styled, even a paragraph buried several levels deep.

### 7. Child selector `>`

The `>` means "only **direct children**", not deeper descendants:

```css
div > p {
  background-color: #880404;
}
```

In the project's markup, a `<div>` contains some direct `<p>` children and also a `<section>` with a `<p>` inside it. The direct children get the red background, but the paragraph inside the `<section>` does not, because it is a grandchild, not a direct child.

```
Descendant vs child, using the project's structure:

  <div>
   ├── <p> I am a Child        ← div > p  YES,  div p  YES
   ├── <p> I am a Child        ← div > p  YES,  div p  YES
   └── <section>
         └── <p> Descendant    ← div > p  NO,   div p  YES
```

### 8. Adjacent sibling selector `+`

Selects an element that comes **immediately after** another, at the same level:

```css
h1 + p {
  font-weight: bold;
}
```

Only the very first `<p>` right after an `<h1>` becomes bold. The second paragraph is not affected.

### 9. General sibling selector `~`

Selects **all** siblings that come after an element (not just the next one):

```css
h2 ~ p {
  color: #860303;
}
```

Every `<p>` that shares a parent with the `<h2>` and appears after it gets the color, even with other elements in between.

```
Sibling selectors compared:

  <h1>
  <p>  ← h1 + p  (only this one, the immediate next)
  <p>

  <h2>
  <p>  ← h2 ~ p  (this one)
  <div>
  <p>  ← h2 ~ p  (and this one too, all later siblings)
```

### 10. Pseudo-class `:`

Selects an element in a certain **state**. The classic is `:hover`, which applies while the mouse is over the element:

```css
a:hover {
  background-color: #14a50e;
  color: #1a1a1a;
}
```

Other common pseudo-classes: `:focus` (an input that is selected), `:first-child`, `:last-child`, `:nth-child()`, `:visited` (a link you already clicked). The grid project uses `:nth-child(even)` to color alternating boxes:

```css
.box:nth-child(even) {
  background-color: #e74c3c;
}
```

### 11. Pseudo-element `::`

Selects a **part** of an element, or creates a virtual element. Written with two colons:

```css
p::first-letter {
  font-weight: bold;
  font-size: 30px;
}
```

This styles just the first letter of every paragraph, like a drop cap. Other pseudo-elements: `::first-line`, `::before`, and `::after` (which create extra content, used in the navbar project).

### 12. Grouping selector `,`

A comma lets one rule apply to several selectors at once, avoiding repetition:

```css
h1,
h2,
h3 {
  color: orange;
}
```

All three heading levels turn orange from a single rule.

```
Selector cheat sheet:

  *              every element
  p              by tag name
  .class         by class attribute (reusable)
  #id            by id attribute (unique)
  [attr="val"]   by attribute
  A B            B inside A (any depth)
  A > B          B that is a direct child of A
  A + B          B immediately after A (siblings)
  A ~ B          all B siblings after A
  A:hover        A in a state
  A::first-letter  a part of A
  A, B           A and B together
```

---

## 5. The Cascade, Specificity, and Inheritance

When several rules target the same element, CSS needs a way to decide which one wins. Three ideas control this.

### The cascade (source order)

If two rules have the **same strength**, the one written **later** wins. CSS reads top to bottom, and later rules override earlier ones.

```css
p { color: blue; }
p { color: red; }   /* this wins, it comes later */
```

### Specificity (how targeted a selector is)

A more specific selector beats a less specific one, regardless of order. Roughly, the ranking from weakest to strongest is:

```
Specificity ranking (weakest → strongest):

  1. Type selectors        p, div, h1          (least specific)
  2. Class / attribute     .card, [type="x"], :hover
  3. ID selectors          #header
  4. Inline style          style="..."         (in the HTML)
  5. !important            color: red !important;  (overrides everything)
                                                 (most specific)
```

This is why, back in the basics project, `.browntext` (a class) beats `p` (a type) even though both set the color. The class is more specific.

### Inheritance (styles passing down)

Some properties automatically pass from a parent to its children. Text-related properties like `color`, `font-family`, and `font-size` are inherited. This is why setting `font-family` on `body` styles the whole page:

```css
body {
  font-family: Arial, Helvetica, sans-serif;
}
```

Every element inside `body` inherits that font unless it sets its own. Layout properties like `margin`, `padding`, and `border` are **not** inherited; each element controls its own.

```
How the browser picks a winner:

  1. Does one rule have !important?        → it wins
  2. Otherwise, compare specificity        → more specific wins
  3. If specificity ties, later rule wins  → source order
  4. If nothing sets it, maybe inherited   → from the parent
  5. If not inherited, use the default     → browser default
```

---

## 6. Colors, Units, and Values

The projects in this folder use a wide range of color formats and units. Here is what they mean.

### Colors

```css
color: brown;                        /* named color (140 built-in names) */
color: orangered;
background-color: #1a1a1a;           /* hex: #RRGGBB (dark gray here) */
background-color: #fff;              /* short hex: #RGB (white) */
background-color: rgb(52, 152, 219); /* red, green, blue (0-255 each) */
background-color: rgba(255, 255, 255, 0.9); /* rgb + alpha (opacity) */
```

- **Named colors** like `brown` and `orangered` are easy but limited.
- **Hex codes** like `#1a1a1a` are the most common. Two digits each for red, green, blue. `#fff` is shorthand for `#ffffff` (white).
- **rgb()** sets each channel from 0 to 255.
- **rgba()** adds a fourth value, the **alpha** (opacity) from 0 (transparent) to 1 (solid). The coming-soon project uses `rgba(255, 255, 255, 0.9)` to make a white box slightly see-through so the gradient behind shows faintly.

### Units

CSS has absolute units (fixed) and relative units (scale with something else).

| Unit | Type | Meaning |
|---|---|---|
| `px` | absolute | pixels, a fixed dot on screen |
| `%` | relative | percent of the parent's size |
| `vw` | relative | 1% of the viewport (screen) width |
| `vh` | relative | 1% of the viewport height |
| `rem` | relative | multiple of the root font size |
| `em` | relative | multiple of the element's font size |
| `fr` | grid only | a fraction of leftover grid space |

Examples pulled from the projects:

```css
width: 300px;        /* fixed 300 pixels */
width: 100%;         /* full width of the parent */
height: 100vh;       /* full height of the screen (used to center things) */
font-size: 2vw;      /* scales with screen width (responsive text) */
padding: 2rem;       /* 2 times the root font size */
```

`100vh` (full viewport height) shows up a lot because it is the trick for making a section fill the whole screen so you can center content in it vertically.

### calc()

`calc()` lets you mix units in a calculation. The responsive project uses it for font size:

```css
font-size: calc(16px + 1vw);
```

This means "16 pixels, plus 1% of the screen width". The text has a solid minimum size but still grows on bigger screens.

---

## 7. The Box Model

**Source:** `04_boxmodel/boxmodel.html`

This is the single most important layout concept in CSS. **Every element on the page is a rectangular box**, and that box is built from four layers, from the inside out:

```
The box model, layer by layer:

  ┌─────────────────────────────────────────────┐
  │  MARGIN  (space OUTSIDE, pushes others away) │
  │  ┌───────────────────────────────────────┐  │
  │  │  BORDER  (the edge line)              │  │
  │  │  ┌─────────────────────────────────┐  │  │
  │  │  │  PADDING  (space inside border) │  │  │
  │  │  │  ┌───────────────────────────┐  │  │  │
  │  │  │  │  CONTENT  (text, images)  │  │  │  │
  │  │  │  └───────────────────────────┘  │  │  │
  │  │  └─────────────────────────────────┘  │  │
  │  └───────────────────────────────────────┘  │
  └─────────────────────────────────────────────┘
```

The box model project lists these clearly:

- **Content**: the actual text or image.
- **Padding**: space between the content and the border. It is *inside* the box, so it shares the background color.
- **Border**: a line drawn around the padding.
- **Margin**: space *outside* the border that pushes other elements away. It is always transparent.

Here is the demo box from the project:

```css
.box-model-demo {
  width: 300px;
  height: 200px;
  padding: 20px;
  border: 5px solid #333;
  margin: 30px;
  background-color: #f0f0f0;
}
```

### The big gotcha: box-sizing

By default, `width` and `height` set the size of the **content only**. Padding and border are added *on top*, making the element bigger than you asked. The project explains this exactly:

```css
.content-box {
  box-sizing: content-box;   /* the DEFAULT */
}
```

With `content-box`, the real width is:

```
width 300 + padding (20 left + 20 right) + border (5 left + 5 right) = 350px
```

So you write `300px` but the box actually takes `350px`. This surprises everyone at first.

The fix is `border-box`:

```css
.border-box {
  box-sizing: border-box;
}
```

With `border-box`, the `width: 300px` **includes** padding and border. The box is exactly 300px wide, and the content shrinks to fit. This is far more predictable, which is why the selectors project (and most real projects) set it on everything:

```css
* {
  box-sizing: border-box;
}
```

```
content-box vs border-box (width: 300px, padding: 20px, border: 5px):

  content-box:  content is 300px  →  total box = 350px  (300 + 40 + 10)
  border-box:   total box is 300px →  content shrinks to 250px

  Rule of thumb: set box-sizing: border-box on everything and
  never fight this math again.
```

### Shorthand for margin and padding

You can set all four sides at once or individually:

```css
padding: 10px;              /* all four sides */
padding: 10px 20px;         /* top+bottom 10, left+right 20 */
padding: 5px 15px;          /* the navbar uses this: 5 vertical, 15 horizontal */
margin: 10px 20px 30px 40px;/* top, right, bottom, left (clockwise) */
```

---

## 8. Display: Block, Inline, and Inline-Block

**Source:** `04_boxmodel/boxmodel.html`, `04_navbar/navbar.html`

The `display` property controls how an element flows and whether width and height apply to it. This is another concept the box model project demonstrates directly.

### Block

Block elements start on a new line and take the **full available width**. `<div>`, `<p>`, and headings are block by default. Width and height work normally:

```css
.block-example {
  width: 200px;
  height: 100px;   /* these are respected */
}
```

### Inline

Inline elements flow along with text and take only as much width as their content. `<span>`, `<a>`, and `<strong>` are inline. The important quirk: **inline elements ignore width and height**. The project points this out in a comment:

```css
.inline-example {
  width: 200px;    /* IGNORED on an inline element */
  height: 100px;   /* IGNORED */
}
```

### Inline-block

`inline-block` is the best of both: the element flows inline (sits next to other things) but **also respects width and height**. The box model project uses it on highlighted spans:

```css
.highlight {
  background-color: yellow;
  display: inline-block;
}
```

The navbar project uses `display: block` on links so they fill their whole list item and become easier to click:

```css
.nav2 a {
  display: block;
  width: 100%;
  padding: 10px;
}
```

```
display values compared:

  block         starts new line, full width, w/h work
                ┌──────────────────────────────┐
                │  block                        │
                └──────────────────────────────┘

  inline        flows in text, w/h IGNORED
                inline inline inline

  inline-block  flows in text BUT w/h work
                [box] [box] [box]

  none          removed from the page entirely (used to hide things)
```

`display: none` completely removes an element from view. The navbar dropdown and the responsive project both use it to hide and show content.

---

## 9. Pseudo-classes and Pseudo-elements

**Source:** `03_selectors/selectors.html`, `04_navbar/navbar.html`

These were introduced as selectors in section 4, but they deserve a closer look because the navbar project uses them for a real effect.

### :hover for interactivity

`:hover` styles an element while the mouse is over it. Almost every button in these projects changes on hover:

```css
button:hover {
  background-color: #0741ac;
}
```

### ::before and ::after create content

These pseudo-elements insert a virtual element before or after an element's content. They need a `content` property to exist, even if it is empty. The navbar uses `::after` to build an animated underline:

```css
.nav1 a::after {
  content: "";              /* required, even when empty */
  position: absolute;
  height: 2px;
  width: 100%;
  bottom: 0;
  left: 0;
  background-color: #0a65c7;
  transform: scaleX(0);     /* start invisible (0 width) */
  transition: transform 0.3s ease-in-out;
}

.nav1 a:hover::after {
  transform: scaleX(1);     /* on hover, grow to full width */
}
```

This creates a thin line under each link that slides out smoothly when you hover. It combines four ideas: a pseudo-element (`::after`), positioning (`absolute`), a transform (`scaleX`), and a transition (the smooth animation). We cover the last two in section 13.

```
The animated underline effect:

  Normal state:   Home           (scaleX(0), line invisible)

  On hover:       Home           (scaleX(1), line grows out)
                  ────

  The transition makes it slide open over 0.3 seconds
  instead of snapping instantly.
```

---

## 10. Positioning

**Source:** `04_navbar/navbar.html`

The `position` property controls how an element is placed. It is essential for overlapping elements, dropdowns, and the underline effect above.

- **`static`**: the default. The element sits in normal flow.
- **`relative`**: positioned relative to where it *would* have been. Also makes it the reference point for absolutely positioned children.
- **`absolute`**: positioned relative to its nearest positioned ancestor (an ancestor with `relative`, `absolute`, or `fixed`). It is taken out of normal flow.
- **`fixed`**: positioned relative to the screen, stays put when you scroll.
- **`sticky`**: acts normal until you scroll past it, then sticks.

The navbar uses the classic `relative` parent plus `absolute` child pattern for the underline:

```css
.nav1 a {
  position: relative;   /* the link becomes the reference point */
}
.nav1 a::after {
  position: absolute;   /* the underline positions itself against the link */
  bottom: 0;            /* stick to the bottom of the link */
  left: 0;
}
```

And the dropdown menu uses `absolute` so it floats over the page instead of pushing other content down:

```css
.nav3 .dropdown:hover .dropdown-content {
  display: block;
  position: absolute;
  background-color: #3e3e3e;
  min-width: 150px;
}
```

```
The relative + absolute pattern:

  ┌─ .nav1 a  (position: relative) ─────────┐
  │  Home                                    │
  │                                          │
  │  ::after (position: absolute, bottom:0)  │
  │  ────────────────────────────────────    │  ← pinned to the bottom
  └──────────────────────────────────────────┘

  The absolute child measures itself against the
  relative parent, not the whole page.
```

---

## 11. Flexbox

**Source:** `06_flexbox/flexbox.html`, plus `02_login_project`, `04_navbar`, `05_coming_soon`

Flexbox (the Flexible Box Layout) is a system for arranging items in **one direction**, either a row or a column. It makes it easy to space items out, center them, and let them grow or shrink. It is used all over these projects and is the tool you will reach for most often.

You turn any element into a **flex container** with `display: flex`. Its direct children become **flex items** that you can then control.

```css
.flex-container {
  display: flex;
}
```

### The two axes

Flexbox has a **main axis** (the direction items flow) and a **cross axis** (perpendicular to it). By default the main axis is horizontal (a row).

```
Flex axes (default row direction):

  cross axis (vertical)
      ▲
      │   ┌────┐ ┌────┐ ┌────┐
      │   │ 1  │ │ 2  │ │ 3  │   →  main axis (horizontal)
      │   └────┘ └────┘ └────┘
      │
  justify-content controls spacing along the MAIN axis
  align-items controls spacing along the CROSS axis
```

### flex-direction: row or column

`flex-direction` sets the main axis. The flexbox project shows both:

```css
.flex-container { flex-direction: row; }    /* default: left to right */
.flex-container { flex-direction: column; } /* top to bottom */
```

### justify-content (spacing along the main axis)

This is how you distribute items along the main direction:

```css
justify-content: center;         /* pack items in the center */
justify-content: space-between;  /* first and last at edges, equal gaps between */
justify-content: space-around;   /* equal space around each item */
```

The navbar uses `space-around` to spread menu links evenly:

```css
.nav1 ul {
  display: flex;
  justify-content: space-around;
  align-items: center;
}
```

```
justify-content options (main axis):

  flex-start      [1][2][3]..............
  center          .......[1][2][3].......
  flex-end        ..............[1][2][3]
  space-between   [1].....[2].....[3]
  space-around    ..[1]....[2]....[3]..
```

### align-items (spacing along the cross axis)

This aligns items across the container. `align-items: center` is the famous trick for **vertical centering**:

```css
.flex-container {
  align-items: center;   /* center items vertically in a row */
}
```

The login and coming-soon projects combine `justify-content: center` and `align-items: center` on a full-height body to perfectly center a card on the screen:

```css
body {
  display: flex;
  justify-content: center;   /* center horizontally */
  align-items: center;       /* center vertically */
  height: 100vh;             /* fill the whole screen so centering is visible */
}
```

This four-line pattern is the single most useful layout snippet in CSS. Memorize it.

### flex-wrap

By default, flex items squeeze onto one line. `flex-wrap: wrap` lets them flow onto new lines when they run out of room:

```css
.flex-container {
  flex-wrap: wrap;
}
```

### align-content

When items wrap onto multiple lines, `align-content` spaces those lines within the container (this only has an effect when there are multiple rows):

```css
.flex-container {
  flex-wrap: wrap;
  align-content: space-around;
}
```

### Item-level properties

Some properties go on the individual items, not the container. The flexbox project demonstrates these with inline styles:

- **`order`**: changes the visual order without changing the HTML. The project reverses 1-5 by giving them orders 5-1:

```css
.circle { order: 5; }  /* higher order appears later */
```

- **`flex-grow`**: how greedily an item grabs leftover space. An item with `flex-grow: 5` takes five times as much extra space as one with `flex-grow: 1`:

```css
.circle { flex-grow: 5; }
```

- **`align-self`**: lets one item override the container's `align-items` for itself:

```css
.circle { align-self: flex-start; }  /* this one item aligns to the top */
```

- **`flex: 1 1 300px`** (a shorthand used in the responsive project) means grow, shrink, and start at a base width of 300px. It is common for responsive cards.

```
Flexbox properties, container vs item:

  ON THE CONTAINER (display: flex)     ON THE ITEMS
  ────────────────────────────────     ──────────────────
  flex-direction   row / column        order       reorder
  justify-content  main-axis spacing   flex-grow   grab space
  align-items      cross-axis align    align-self  override align
  flex-wrap        wrap / nowrap        flex        grow/shrink/basis
  align-content    multi-line spacing
```

---

## 12. CSS Grid

**Source:** `07_grid/gridbasics.html`, `07_grid/gridMasterClass.html`

Where flexbox lays out items in **one direction**, CSS Grid lays them out in **two directions** at once (rows and columns). It is the right tool for full-page layouts and any grid of items. You create a grid with `display: grid`.

### Defining rows and columns

`grid-template-columns` and `grid-template-rows` define the structure. From `gridbasics.html`:

```css
.grid-container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);   /* 3 equal columns */
  grid-template-rows: 100px 200px 100px;   /* 3 rows of set heights */
  gap: 4px;                                /* space between cells */
}
```

Two key ideas here:

- **`fr` (fraction unit)**: `1fr` means "one share of the leftover space". `repeat(3, 1fr)` makes three columns that split the width equally. `1fr 2fr 1fr` (from the masterclass) makes the middle column twice as wide as the outer two.
- **`repeat(3, 1fr)`**: shorthand for `1fr 1fr 1fr`. Saves typing when columns repeat.
- **`gap`**: the space between grid cells. Cleaner than adding margins to every item.

```
grid-template-columns: repeat(3, 1fr) makes:

  ┌───────┬───────┬───────┐
  │ 1fr   │ 1fr   │ 1fr   │   each column takes an equal share
  ├───────┼───────┼───────┤
  │       │       │       │
  └───────┴───────┴───────┘

  grid-template-columns: 1fr 2fr 1fr makes:

  ┌─────┬───────────┬─────┐
  │ 1fr │   2fr     │ 1fr │   middle column is twice as wide
  └─────┴───────────┴─────┘
```

### Placing items across cells

Items can span multiple columns or rows using `grid-column` and `grid-row` with line numbers. From `gridbasics.html`:

```css
.item1 {
  grid-column: 1/3;   /* start at line 1, end at line 3 (spans 2 columns) */
  grid-row: 1/2;      /* occupies row 1 */
}
.item2 {
  grid-column: 3/4;
  grid-row: 1/3;      /* spans 2 rows tall */
}
```

Grid lines are numbered starting at 1 on the left/top edge. `1/3` means "from line 1 to line 3", which covers the two columns in between.

```
Grid line numbers (for a 3-column grid):

  line1   line2   line3   line4
    │       │       │       │
    ▼       ▼       ▼       ▼
    ┌───────┬───────┬───────┐
    │  item spanning 1/3    │  item │
    │  (columns 1 and 2)    │  3/4  │
    └───────────────────────┴───────┘

  grid-column: 1/3  →  start line 1, stop before line 3
```

### grid-template-areas (named layout)

The masterclass shows a powerful feature: naming regions and drawing your layout as text. You assign each item a name, then map the names into a grid picture:

```css
.grid-container {
  grid-template-areas:
    'header header header'
    'sidebar main main'
    'footer footer footer';
}
```

```css
.box { grid-area: header; }   /* this box goes wherever "header" is */
```

```
The named layout renders as:

  ┌──────────────────────────────┐
  │  header   header   header    │
  ├────────┬─────────────────────┤
  │ sidebar │  main    main      │
  ├────────┴─────────────────────┤
  │  footer   footer   footer    │
  └──────────────────────────────┘

  You literally draw the page with words. Very readable.
```

### Responsive grids without media queries

The masterclass shows a modern trick for grids that reflow on their own:

```css
grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
```

- **`minmax(100px, 1fr)`**: each column is at least 100px, but grows to fill space.
- **`auto-fit`**: fit as many columns as will fit, then wrap the rest to a new row.

Together these make a grid that automatically adds or removes columns as the screen resizes, with no media queries needed.

```
Flexbox vs Grid, when to use which:

  FLEXBOX  →  one direction (a row OR a column)
              navbars, button groups, centering, toolbars

  GRID     →  two directions (rows AND columns together)
              page layouts, photo galleries, card grids, dashboards

  They work together: a grid page with flexbox inside each cell.
```

---

## 13. Transitions and Transforms

**Source:** `04_navbar/navbar.html`, `05_coming_soon/style.css`

### Transitions (smooth changes)

A **transition** makes a property change gradually over time instead of instantly. Without it, a hover color change snaps immediately. With it, the change eases in. The coming-soon button transitions its background:

```css
button {
  background-color: #8a2be2;
  transition: background-color 0.3s ease-in-out;
}
button:hover {
  background-color: #7a1dd1;
}
```

The transition syntax is: `property duration timing-function`.

- **property**: what to animate (`background-color`, or `all` for everything).
- **duration**: how long (`0.3s`).
- **timing-function**: the speed curve (`ease-in-out` starts slow, speeds up, ends slow).

### Transforms (visual changes)

A **transform** changes an element's shape or position visually without affecting the layout around it. The navbar underline uses `scaleX`:

```css
transform: scaleX(0);   /* squished to zero width */
transform: scaleX(1);   /* full width */
```

Common transforms:

```css
transform: scaleX(1);          /* stretch horizontally */
transform: scale(1.2);         /* grow to 120% */
transform: rotate(45deg);      /* spin */
transform: translateX(20px);   /* slide right 20px */
```

Transforms are cheap for the browser to animate, so pairing a `transform` with a `transition` (as the navbar does) gives smooth, high-performance effects.

```
Transition + transform working together (the underline):

  time 0.0s    scaleX(0)   |            (line invisible)
  time 0.15s   scaleX(0.5) |──────      (growing)
  time 0.3s    scaleX(1)   |───────────  (full width)

  The transition spreads the transform change across 0.3 seconds.
```

---

## 14. Backgrounds, Gradients, and Shadows

**Source:** `05_coming_soon/style.css`, `02_login_project/login.html`

### Gradients

A gradient is a smooth blend between colors, used as a background. The coming-soon page uses a diagonal gradient:

```css
.container {
  background: linear-gradient(135deg, #8a2be2, #4169e1);
}
```

`linear-gradient(angle, color1, color2)` blends from the first color to the second along the given angle. `135deg` points to the bottom-right, so the purple fades into blue diagonally.

### Box shadows

`box-shadow` adds depth by drawing a shadow behind an element. The coming-soon button uses one:

```css
box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
```

The values are: horizontal offset, vertical offset, blur radius, and color. `0 4px 6px` means no horizontal shift, 4px down, 6px of blur. Using `rgba` with low opacity makes a soft, realistic shadow.

### Border-radius

`border-radius` rounds the corners of a box. It appears throughout the projects:

```css
border-radius: 8px;    /* rounded corners on the login card */
border-radius: 50%;    /* a perfect circle (used on the flexbox circles) */
```

Setting `border-radius: 50%` on a square element turns it into a circle, which is how the flexbox project makes its round pastel dots:

```css
.circle {
  width: 50px;
  height: 50px;
  border-radius: 50%;
}
```

### object-fit

When an image must fit a set size without distorting, `object-fit` controls how. The coming-soon logo uses `contain`:

```css
.logo {
  width: 150px;
  object-fit: contain;   /* fit the whole image inside, keep proportions */
}
```

`contain` fits the entire image inside the box, while `cover` fills the box and crops the overflow.

### Importing fonts

The coming-soon CSS pulls a Google Font using `@import` at the very top of the file:

```css
@import url("https://fonts.googleapis.com/css2?family=Noto+Sans...");

body {
  font-family: "Noto Sans", sans-serif;
}
```

`@import` must be the first thing in the file. After importing, you use the font name in `font-family`, with `sans-serif` as a fallback in case the font fails to load.

---

## 15. Responsive Design and Media Queries

**Source:** `08_responsive/breakPoint.html`, `08_responsive/Responsive-design.html`

**Responsive design** means a page that looks good on any screen size, from a phone to a wide monitor. The main tool is the **media query**, which applies CSS only when the screen meets a condition.

### The viewport meta tag comes first

Responsive design does not work without this line in the HTML `<head>`, present in every file here:

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

It tells phones to use their real width instead of pretending to be a desktop. Without it, media queries are ignored on mobile.

### Media queries

A media query wraps CSS in a condition. The `breakPoint.html` project changes a div's color at different screen widths:

```css
/* phones: 600px wide or less */
@media (max-width: 600px) {
  .responsive-div {
    background-color: #3498db;
  }
}

/* tablets: between 601px and 1024px */
@media (min-width: 601px) and (max-width: 1024px) {
  .responsive-div {
    background-color: #e74c3c;
  }
}
```

- **`max-width: 600px`** means "apply when the screen is 600px or narrower".
- **`min-width: 601px`** means "apply when the screen is 601px or wider".
- You can combine them with `and` for a range.

The width values where the layout changes are called **breakpoints**. Common ones are around 600px (phones) and 1024px (tablets).

```
How media queries switch styles at breakpoints:

  screen width:  0 ──────── 600 ──────── 1024 ──────── ∞
                 │  phone    │  tablet     │  desktop
                 │  (blue)   │  (red)      │  (default purple)
                 ▼           ▼             ▼
  max-width:600  ████████████
  601 to 1024                 ██████████████
  (base rule)    ═════════════════════════════════════
```

### Fluid layouts

Instead of fixed pixel widths, responsive layouts use percentages and `max-width` so they shrink on small screens but do not get too wide on large ones. From `breakPoint.html`:

```css
.responsive-div {
  width: 100%;         /* fill the available width */
  max-width: 1200px;   /* but never exceed 1200px */
}
```

This is a key pattern: `width: 100%` plus a `max-width` gives you a box that is fluid on phones and capped on desktops.

### Responsive images

Making an image scale with its container is simple:

```css
img {
  width: 100%;
  height: auto;   /* keep proportions, no stretching */
}
```

### Responsive layouts with flex and grid

The `Responsive-design.html` project shows that flexbox and grid are responsive tools in themselves. Flex items with a base size wrap automatically:

```css
.box {
  flex: 1 1 300px;   /* grow, shrink, prefer 300px; wraps when cramped */
}
```

And a media query can restructure a grid at each breakpoint:

```css
.responsive-container {
  grid-template-columns: repeat(3, 1fr);   /* 3 columns by default */
}
@media (max-width: 800px) {
  .responsive-container {
    grid-template-columns: repeat(2, 1fr); /* 2 columns on tablets */
  }
}
@media (max-width: 500px) {
  .responsive-container {
    grid-template-columns: 1fr;            /* 1 column on phones */
  }
}
```

### Show and hide by screen size

Media queries can hide or reveal elements. The responsive project toggles content for mobile:

```css
@media (max-width: 600px) {
  .hidden-on-mobile { display: none; }
  .visible-on-mobile { display: block; }
}
```

### Responsive typography

Text can scale with the screen using viewport units or `calc()`:

```css
font-size: 2vw;                 /* grows with screen width */
font-size: calc(16px + 1vw);    /* a fixed base plus a scaling part */
```

```
The responsive toolkit:

  meta viewport      → makes responsive possible at all (required)
  %  and max-width   → fluid boxes that cap on large screens
  media queries      → change styles at breakpoints
  flex-wrap / grid   → layouts that reflow on their own
  vw / calc()        → text that scales with the screen
  width:100%;h:auto  → images that shrink to fit
```

---

## 16. Project Walkthrough: Login Card

**Source:** `02_login_project/login.html`

This small project ties together centering, the box model, forms styling, and hover states. Here is how it works, piece by piece.

**Center the card on the screen.** The body becomes a full-height flex container that centers its child both ways:

```css
body {
  margin: 0;
  height: 100vh;
  display: flex;
  justify-content: center;   /* horizontal center */
  align-items: center;       /* vertical center */
  background-color: #1a1a1a;
}
```

Setting `margin: 0` removes the browser's default body margin, and `height: 100vh` makes the body fill the screen so the centering is actually visible.

**Style the card.** Padding, rounded corners, and a fixed width give it shape:

```css
.card {
  background-color: #d5d4d4;
  border-radius: 8px;
  padding: 30px;
  width: 300px;
}
```

**Stack the form fields vertically.** The form is itself a flex container in column direction, so inputs and the button stack neatly:

```css
form {
  display: flex;
  flex-direction: column;
}
```

**Style inputs and the button, with a hover effect:**

```css
input {
  margin-bottom: 15px;
  padding: 10px;
  border-radius: 5px;
  border: 1px solid black;
}
button:hover {
  background-color: #0741ac;   /* darkens when hovered */
}
```

```
The login layout:

  ┌──────────── body (100vh, flex, centered) ────────────┐
  │                                                       │
  │            ┌──── .card (300px) ────┐                  │
  │            │        Login          │                  │
  │            │  ┌─────────────────┐  │                  │
  │            │  │ email input     │  │  form is a       │
  │            │  ├─────────────────┤  │  column flexbox  │
  │            │  │ password input  │  │                  │
  │            │  ├─────────────────┤  │                  │
  │            │  │   Login button  │  │                  │
  │            │  └─────────────────┘  │                  │
  │            └───────────────────────┘                  │
  │                                                       │
  └───────────────────────────────────────────────────────┘
```

The coming-soon page (`05_coming_soon`) uses the exact same centering pattern, just with a gradient background and a Google Font.

---

## 17. Project Walkthrough: Navbars

**Source:** `04_navbar/navbar.html`

This project builds three navigation bar styles, each teaching something different. All three start from a list of links, because a `<ul>` of `<li>` is the standard, accessible way to build a menu.

### Removing list defaults

Every navbar first strips the bullets and default spacing off the list:

```css
.nav1 ul {
  list-style-type: none;   /* remove bullet points */
  margin: 0;
  padding: 0;
}
```

### Variation 1: horizontal with animated underline

The links sit in a row using flexbox, and each gets the `::after` underline effect from section 9:

```css
.nav1 ul {
  display: flex;
  justify-content: space-around;
  align-items: center;
}
```

### Variation 2: vertical sidebar

No flexbox here. The list stays in its default vertical block flow, and links are set to `display: block` so the whole row is clickable:

```css
.nav2 a {
  display: block;
  width: 100%;
  padding: 10px;
}
.nav2 a:hover {
  background-color: #137e0a;
}
```

### Variation 3: horizontal with a dropdown

A flexbox row again, but one item has a hidden submenu that appears on hover. The submenu is hidden by default and revealed with `display: block` when its parent is hovered. It uses `position: absolute` so it floats over the page instead of pushing content down:

```css
.nav3 .dropdown-content {
  display: none;                 /* hidden by default */
}
.nav3 .dropdown:hover .dropdown-content {
  display: block;                /* shown when the parent is hovered */
  position: absolute;
  background-color: #3e3e3e;
  min-width: 150px;
}
```

```
The three navbar patterns:

  Variation 1 (flex row, underline):
    Home   About   Products   [Login]
    ────                              (line slides out on hover)

  Variation 2 (block column):
    ┌──────────┐
    │ Home     │
    │ About    │  ← full-width clickable rows
    │ Products │
    │ [Login]  │
    └──────────┘

  Variation 3 (flex row, dropdown):
    Home   About ▼   Products   [Login]
                │
                ├─ Our Story    ← appears on hover,
                ├─ Team           floats via position:absolute
                └─ Contact
```

---

## 18. Bootstrap: A CSS Framework

**Source:** `09_bootstrap/bootstrap.html`

**Bootstrap** is a pre-written CSS library. Instead of writing every style yourself, you add ready-made **class names** to your HTML and Bootstrap styles them for you. It is a fast way to build clean, responsive layouts without writing much CSS.

### Adding Bootstrap

You link Bootstrap's CSS from a CDN (a public web address) in the `<head>`, and its JavaScript bundle before `</body>`:

```html
<head>
  <link rel="stylesheet"
        href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" />
</head>
<body>
  ...
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
```

The CSS handles the looks; the JavaScript handles interactive parts like the collapsing mobile menu.

### Using utility and component classes

With Bootstrap linked, you style by adding classes. The project builds a navbar and a card entirely from Bootstrap classes:

```html
<nav class="navbar navbar-expand-lg bg-dark">
  <div class="container-fluid">
    <a class="navbar-brand" href="#">ChaiCode</a>
    ...
  </div>
</nav>

<div class="card" style="width: 18rem">
  <img src="..." class="card-img-top" alt="..." />
  <div class="card-body bg-primary text-white">
    <h5 class="card-title">Card title</h5>
    <p class="card-text">Some quick example text.</p>
    <a href="#" class="btn btn-primary">Go somewhere</a>
  </div>
</div>
```

Notice you write almost no CSS of your own. Classes like these do the work:

| Bootstrap class | What it does |
|---|---|
| `navbar`, `navbar-expand-lg` | responsive nav that collapses on small screens |
| `container-fluid` | full-width centered container |
| `card`, `card-body`, `card-title` | a pre-styled card component |
| `btn`, `btn-primary` | a styled button |
| `bg-dark`, `bg-primary` | background colors |
| `text-white` | white text |
| `d-flex` | `display: flex` as a utility class |
| `me-2`, `mb-2` | margin helpers (margin-end, margin-bottom) |

### The tradeoff

```
Writing CSS yourself vs using Bootstrap:

  Your own CSS          Bootstrap
  ─────────────         ─────────────────
  full control          fast to build
  smaller files         consistent, tested, responsive out of the box
  more time             less unique (many sites look alike)
  you learn CSS         you rely on class names

  Best approach: learn CSS first (sections 1-17), THEN use
  frameworks like Bootstrap to move quickly when it helps.
```

Bootstrap is built on the same flexbox, grid, and box-model ideas you already learned. It is a shortcut, not a replacement for understanding CSS.

---

## 19. Quick Reference — Mental Model

```
CSS — how everything in this folder connects:

  ┌────────────────────────────────────────────────────────────────────┐
  │  ADD CSS:  inline  <  internal  <  external (.css file, preferred)  │
  │  RULE:     selector { property: value; }                            │
  └───────────────────────────────┬────────────────────────────────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │  SELECTING       │  │  THE BOX MODEL   │  │  VALUES              │
  │  * type .class   │  │  content         │  │  colors: hex, rgba   │
  │  #id [attr]      │  │  padding         │  │  units: px % vw rem  │
  │  A B  A>B  A+B   │  │  border          │  │  fr (grid), calc()   │
  │  A~B :hover ::   │  │  margin          │  │                      │
  │  specificity     │  │  box-sizing!     │  │                      │
  └──────────────────┘  └──────────────────┘  └──────────────────────┘
            │                     │                     │
            ▼                     ▼                     ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │  LAYOUT          │  │  LOOKS           │  │  RESPONSIVE          │
  │  display         │  │  gradients       │  │  meta viewport       │
  │  FLEXBOX (1D)    │  │  box-shadow      │  │  @media queries      │
  │  GRID (2D)       │  │  border-radius   │  │  % + max-width       │
  │  position        │  │  transition      │  │  flex-wrap / grid    │
  │                  │  │  transform       │  │  vw / calc() text    │
  └──────────────────┘  └──────────────────┘  └──────────────────────┘
            │
            ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  BUILD FASTER:  Bootstrap (pre-made classes, same ideas)     │
  └──────────────────────────────────────────────────────────────┘

  Golden patterns to memorize:
    • Center anything:  display:flex; justify-content:center;
                        align-items:center; height:100vh;
    • Predictable sizing:  * { box-sizing: border-box; }
    • Fluid box:  width:100%; max-width:1200px;
    • Circle:  border-radius:50%; on a square
```

### Cheat sheet of everything used in this folder

| Property / concept | Project it appears in | What it does |
|---|---|---|
| `color`, `background-color` | all | text and background color |
| `font-size`, `font-family` | most | text size and typeface |
| `padding`, `margin`, `border` | boxmodel, all | the box model spacing |
| `box-sizing: border-box` | selectors, boxmodel | include padding/border in width |
| `.class`, `#id`, `*`, `[attr]` | selectors | ways to target elements |
| `A > B`, `A + B`, `A ~ B`, `A B` | selectors | relationship selectors |
| `:hover`, `:nth-child`, `::after` | selectors, navbar, grid | states and pseudo-elements |
| `display: flex` | login, navbar, flexbox | one-dimensional layout |
| `justify-content`, `align-items` | login, flexbox | flex spacing and alignment |
| `flex-direction`, `flex-wrap` | flexbox | flex flow direction and wrapping |
| `flex-grow`, `order`, `align-self` | flexbox | per-item flex control |
| `display: grid` | grid | two-dimensional layout |
| `grid-template-columns`, `gap` | grid | grid structure and spacing |
| `repeat()`, `fr`, `minmax()`, `auto-fit` | grid | flexible grid sizing |
| `grid-column`, `grid-area` | grid | placing items in the grid |
| `position: relative / absolute` | navbar | precise placement, overlaps |
| `transition`, `transform` | navbar, coming_soon | smooth animated changes |
| `linear-gradient`, `box-shadow` | coming_soon | rich backgrounds and depth |
| `border-radius`, `object-fit` | login, flexbox, coming_soon | rounded corners, image fit |
| `@import` | coming_soon | load a web font |
| `@media (max-width: ...)` | responsive | apply styles at breakpoints |
| `100vh`, `vw`, `calc()`, `%` | login, responsive | relative sizing units |
| Bootstrap classes | bootstrap | pre-made styling via class names |

---

*All examples taken directly from the `css/` course files: `01_basics`, `02_login_project`, `03_selectors`, `04_boxmodel`, `04_navbar`, `05_coming_soon`, `06_flexbox`, `07_grid`, `08_responsive`, and `09_bootstrap`. A few extra concepts (positioning details, specificity ranking, shorthand units) were added where the folder leaves gaps.*
