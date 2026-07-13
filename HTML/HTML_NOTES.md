# HTML — Complete Guide

> Based on the code in `code+files/html/` — every concept explained with context, not just syntax. Written in plain English with extra concepts added where the folder skips them.

---

## Table of Contents

1. [What Is HTML?](#1-what-is-html)
2. [The Skeleton of Every Page](#2-the-skeleton-of-every-page)
3. [The `<head>` and Metadata](#3-the-head-and-metadata)
4. [Tags, Elements, and Attributes](#4-tags-elements-and-attributes)
5. [Headings and Paragraphs](#5-headings-and-paragraphs)
6. [Text Formatting Elements](#6-text-formatting-elements)
7. [Links (Anchor Tags)](#7-links-anchor-tags)
8. [Images](#8-images)
9. [Lists](#9-lists)
10. [Block vs Inline Elements](#10-block-vs-inline-elements)
11. [Tables](#11-tables)
12. [Forms](#12-forms)
13. [Semantic HTML](#13-semantic-html)
14. [Media: Figures, Video, and Audio](#14-media-figures-video-and-audio)
15. [HTML Entities](#15-html-entities)
16. [Good Habits and Common Mistakes](#16-good-habits-and-common-mistakes)
17. [Quick Reference — Mental Model](#17-quick-reference--mental-model)

---

## 1. What Is HTML?

HTML stands for **HyperText Markup Language**. It is the language used to describe the **structure and content** of a web page. When you open any website, your browser is reading an HTML file and turning it into the text, images, buttons, and links you see on screen.

The word "markup" is the key idea. HTML does not run logic like JavaScript, and it does not decide colors and layout like CSS. Instead, you take plain content and wrap it in **tags** that label what each piece of content *is*. A tag says "this is a heading", "this is a paragraph", "this is a link", and the browser knows how to display each kind.

```
The three languages of the web and what each one does:

  HTML   →  Structure and content   (the words, images, and layout of parts)
  CSS    →  Presentation and style  (colors, spacing, fonts, positioning)
  JS     →  Behavior and logic      (respond to clicks, change the page live)

  Think of a house:
    HTML = the walls, rooms, and doors  (what exists and where)
    CSS  = the paint, furniture, decor  (how it looks)
    JS   = the electricity and plumbing (how it reacts and works)
```

An HTML file is just a text file that ends in `.html`. The browser reads it from top to bottom and builds the page. You can open any `.html` file in this folder directly in a browser to see the result.

---

## 2. The Skeleton of Every Page

**Source:** `index.html`, and the top of every file in this folder

Every HTML document follows the same basic skeleton. Here is the smallest complete page, taken from `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <h1>Hello from Hitesh from web dev course</h1>
</body>
</html>
```

Let us break down each part, because these lines appear at the top of every single file in the folder.

- **`<!DOCTYPE html>`** tells the browser "this page uses modern HTML5". It is not really a tag, it is a declaration, and it must always be the very first line. Without it, browsers switch to an old "quirks mode" that behaves in strange, outdated ways.

- **`<html lang="en">`** is the root element. Everything else lives inside it. The `lang="en"` attribute says the page is in English, which helps screen readers pronounce words correctly and helps search engines and translation tools.

- **`<head>`** holds information *about* the page that the visitor does not see directly: the title, character settings, links to stylesheets, and so on. More on this in section 3.

- **`<body>`** holds everything the visitor actually sees: headings, text, images, links, forms, and all other visible content.

```
The nesting structure of every page:

  <!DOCTYPE html>          ← "I am modern HTML"
  <html>                   ← the root, wraps everything
  ├── <head>               ← invisible info about the page
  │     ├── <meta>         ← settings (charset, viewport)
  │     └── <title>        ← text shown on the browser tab
  └── <body>               ← everything the user SEES
        ├── <h1>
        ├── <p>
        └── ...
```

Notice how tags come in pairs: `<body>` opens and `</body>` closes. The slash `/` marks the closing tag. Content goes between the opening and closing tags. Tags nest inside each other like boxes inside boxes, and they should always close in the reverse order they opened.

> **A note on the source:** in `index.html` the heading text is written across three lines. In the browser it shows up as a single line, "Hello from Hitesh from web dev course". This is because HTML collapses any run of spaces, tabs, or new lines into a single space. Where you break lines in your code does not affect the output. Only tags change the layout.

---

## 3. The `<head>` and Metadata

**Source:** the `<head>` section of every file

The `<head>` is where you put **metadata**, which means data about the page rather than page content. Nothing inside `<head>` shows up in the main window (except the title, which appears on the browser tab). Here is the standard head used throughout the folder:

```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tables</title>
</head>
```

- **`<meta charset="UTF-8">`** sets the character encoding to UTF-8. This is what lets the page correctly display letters from every language, plus symbols and emoji. Always include it, and always near the top.

- **`<meta name="viewport" content="width=device-width, initial-scale=1.0">`** controls how the page fits on phones and tablets. `width=device-width` tells the browser to match the page width to the actual screen width, and `initial-scale=1.0` sets the starting zoom level to normal. Without this line, mobile browsers pretend to be a wide desktop and shrink everything, making text tiny. This single line is essential for responsive, mobile-friendly pages.

- **`<title>`** sets the text shown on the browser tab and used as the name when someone bookmarks the page. Each file in the folder gives it a different value, like "Tables", "forms", or "Gym workout", which is why the tab label changes when you open each one.

```
Where each part of the <head> shows up:

  ┌─────────────────────────────────────────┐
  │  [ Tables ]  ← the <title> shows here    │   Browser tab
  ├─────────────────────────────────────────┤
  │                                          │
  │   The <body> content shows here.         │   Page window
  │   charset and viewport work silently     │
  │   in the background.                     │
  │                                          │
  └─────────────────────────────────────────┘
```

Other common things that go in the head (not used in this folder but worth knowing): `<meta name="description" content="...">` for the summary search engines show, `<link rel="stylesheet" href="style.css">` to attach CSS, and `<link rel="icon" href="favicon.ico">` for the little tab icon.

---

## 4. Tags, Elements, and Attributes

These three words get used a lot, so it helps to be precise about them.

- A **tag** is the thing in angle brackets, like `<p>` or `</p>`.
- An **element** is the whole unit: the opening tag, the content, and the closing tag together. `<p>Hello</p>` is one paragraph element.
- An **attribute** is extra information added inside the opening tag, written as `name="value"`. Attributes configure how an element behaves or looks.

```
Anatomy of an element:

     opening tag         closing tag
        │                    │
        ▼                    ▼
     <a href="https://chaicode.com"> go to chaicode </a>
      │  └──── attribute ───┘         └── content ──┘
      │        name  value
    element
    name
```

You can see real attributes throughout the folder. From `01_intro.html`:

```html
<h1 title="chaicode">Lorem ipsum dolor sit amet consectetur.</h1>
<h2 title="new tooltip">Lorem ipsum dolor sit amet.</h2>
```

The `title` attribute here creates a **tooltip**: hover your mouse over the heading and a small box appears showing the text "chaicode". The `title` attribute works on almost any element and is a nice way to add hover hints.

Some elements do not wrap content and have no closing tag. These are called **empty** or **self-closing** elements. `<img>`, `<meta>`, `<input>`, and `<br>` are examples. They carry all their information in attributes:

```html
<img src="./images/hc.png" width="300px" alt="">
<meta charset="UTF-8">
```

---

## 5. Headings and Paragraphs

**Source:** `01_intro.html`, `02_headings_list.html`

### Headings

HTML gives you six levels of heading, from `<h1>` (most important, largest) down to `<h6>` (least important, smallest). They create a visual and structural hierarchy for your content.

```html
<h1 title="chaicode">Lorem ipsum dolor sit amet consectetur.</h1>
<h2 title="new tooltip">Lorem ipsum dolor sit amet.</h2>
<h4>Lorem ipsum dolor sit amet consectetur adipisicing elit...</h4>
```

```
The heading scale (size shrinks as the number grows):

  <h1>  Biggest — the main title of the page
  <h2>    Section titles
  <h3>      Sub-sections
  <h4>        Smaller sub-sections
  <h5>          Even smaller
  <h6>            Smallest
```

Two important rules about headings:

1. **Use only one `<h1>` per page.** It represents the single main topic. This helps search engines and screen readers understand what the page is about.
2. **Do not skip levels for looks.** Headings describe structure, not size. Notice that `01_intro.html` jumps from `<h2>` straight to `<h4>`, skipping `<h3>`. It still displays fine, but it breaks the logical outline. If you want smaller text, use CSS, do not misuse a lower heading level. Go `h1` then `h2` then `h3` in order.

### Paragraphs

The `<p>` tag marks a paragraph of text. Browsers automatically add space above and below each paragraph to separate them.

```html
<p>Lorem ipsum dolor, sit amet consectetur adipisicing elit. Sunt sequi
nostrum facilis id inventore animi earum aut...</p>
```

Remember from section 2 that HTML collapses whitespace. No matter how many spaces or line breaks you put inside a `<p>`, the browser shows the text as one flowing block. To force a line break inside a paragraph, use the empty `<br>` element. To create separate blocks of text, use separate `<p>` elements.

> **What is "Lorem ipsum"?** It is placeholder text, sometimes called "dummy text", that designers use to fill space while building a layout before the real words are ready. It looks like Latin but means nothing. You will see it all over this folder.

---

## 6. Text Formatting Elements

**Source:** `02_headings_list.html`, `05_semantics.html`

Inside your text you often want to emphasize or mark up small parts. HTML has elements for this that wrap just a few words rather than a whole block.

```html
<p>Lorem ipsum <em>dolor</em> sit.</p>
```

Here `<em>` wraps a single word. It stands for **emphasis** and browsers show it in *italics*. But the meaning matters more than the look: `<em>` tells screen readers to stress that word when reading aloud.

Here are the common inline text elements:

| Element | Meaning | Default look |
|---|---|---|
| `<em>` | emphasis (stress a word) | *italic* |
| `<strong>` | strong importance | **bold** |
| `<i>` | alternate voice (idiom, term) | *italic* |
| `<b>` | draw attention, no extra importance | **bold** |
| `<code>` | a piece of computer code | `monospace font` |
| `<mark>` | highlighted text | yellow highlight |
| `<small>` | fine print | smaller text |
| `<br>` | a line break | moves to next line |

The folder lists these in `02_headings_list.html` when it groups example elements, and it uses `<code>` in `05_semantics.html`:

```html
<code>console.log("hitesh")</code>
```

The `<code>` element displays text in a monospace (fixed-width) font, which is the standard way to show code inside normal text so it stands out from the surrounding words.

> **Meaning over appearance:** prefer `<em>` and `<strong>` over `<i>` and `<b>`. Both pairs look the same, but `<em>` and `<strong>` also carry meaning that assistive technology and search engines understand. Use `<i>` and `<b>` only when you want the visual style without implying importance.

---

## 7. Links (Anchor Tags)

**Source:** `01_intro.html`, `05_semantics.html`

Links are what make the web a web. The **anchor** element `<a>` creates a clickable link to another page, another site, or another spot on the same page. The destination goes in the `href` attribute, which stands for "hypertext reference".

From `01_intro.html`:

```html
<a href="https://chaicode.com"> go to chaicode </a>
<a href="./index.html">go to index</a>
```

These two links show the two main kinds of link destinations:

- **Absolute URL**: `https://chaicode.com` is a full web address pointing to a completely different website. Use the full `https://...` form for external sites.
- **Relative URL**: `./index.html` points to a file relative to the current file. The `./` means "in the same folder as this file". This is how you link between your own pages.

```
Reading a relative path:

  ./index.html        → index.html in the SAME folder
  ./images/hc.png     → hc.png inside the "images" subfolder
  ../about.html       → about.html one folder UP
  /contact.html       → contact.html from the site's root folder
```

### Special href values

In `05_semantics.html` the navigation links use `href="#"`:

```html
<li><a href="#">Home</a></li>
<li><a href="#">About Us</a></li>
```

A lone `#` is a placeholder link that goes nowhere (it jumps to the top of the current page). Developers use it while building, before the real destinations exist. When a page has sections with `id` attributes, you can link to them with `#sectionName` to jump straight to that part.

### Opening links in a new tab

A common addition (not in the folder but worth knowing): add `target="_blank"` to open the link in a new browser tab. When you do this for external sites, also add `rel="noopener noreferrer"` for security:

```html
<a href="https://chaicode.com" target="_blank" rel="noopener noreferrer">
  go to chaicode
</a>
```

---

## 8. Images

**Source:** `01_intro.html`, `02_headings_list.html`, `media.html`

The `<img>` element places an image on the page. It is an empty element (no closing tag), and it needs a `src` attribute telling the browser where the image file is.

```html
<img src="./images/hc.png" width="300px" alt="">
```

Key attributes:

- **`src`** (source): the path to the image file. Just like links, this can be relative (`./images/hc.png`) or a full URL. The images in this folder live in the `images/` subfolder.
- **`alt`** (alternative text): a text description of the image. This is important and often skipped. It is read aloud by screen readers for blind users, and it is shown in place of the image if the file fails to load. In `01_intro.html` the `alt=""` is left empty, which is a missed opportunity. Compare it with `media.html`, which does it right:

```html
<img src="./images/hc.png" title="Hitesh" width="200px" alt="profile photo" />
```

- **`width`** and **`height`**: set the display size. You can use pixels (`200px` or just `200`). If you set only one, the image keeps its proportions. Setting both can stretch the image.

```
Why alt text matters:

  Image loads fine        → the picture shows, alt text stays hidden
  Image fails to load     → the browser shows the alt text instead
  Screen reader user      → hears the alt text read aloud
  Search engine           → reads alt text to understand the image

  Rule: every meaningful image needs a real alt description.
  Decorative images (that add no info) can use alt="".
```

> **Best practice:** always write a real, descriptive `alt` value for images that carry meaning. Describe what the image shows, not "image of...". The empty `alt=""` in `01_intro.html` should really say something like `alt="ChaiCode logo"`.

---

## 9. Lists

**Source:** `02_headings_list.html`

HTML has two main list types, and they are used constantly for menus, steps, and grouped items.

### Ordered list `<ol>`

Use an ordered list when the **order matters**, like steps in a recipe or ranked items. Each item is a list item `<li>`. The browser adds numbers automatically.

```html
<ol>
    <li>orange tea</li>
    <li>black tea</li>
</ol>
```

This displays as:

```
  1. orange tea
  2. black tea
```

### Unordered list `<ul>`

Use an unordered list when the **order does not matter**, like a set of features or a navigation menu. The browser adds bullet points.

```html
<ul>
    <li>heading h1 to h6</li>
    <li>ol, li</li>
    <li>p</li>
</ul>
```

This displays as:

```
  • heading h1 to h6
  • ol, li
  • p
```

```
Choosing between the two:

  Order carries meaning?     →  <ol>   (1, 2, 3...)
  Just a group of items?     →  <ul>   (bullets)

  Both use <li> for each item. Both can be nested inside
  each other to create sub-lists.
```

The only difference between `<ol>` and `<ul>` is the meaning and the default marker (numbers vs bullets). Both hold `<li>` items, and both can be nested to build multi-level lists. Lists are also the standard building block for navigation menus, which you will see in the semantics section.

There is a third, less common list type: the **description list** `<dl>`, which pairs terms `<dt>` with descriptions `<dd>`. Use it for glossaries and key-value pairs.

---

## 10. Block vs Inline Elements

**Source:** `02_headings_list.html`

This is one of the most important ideas for understanding how HTML lays out on the page. Every element is either **block-level** or **inline**, and this decides how it flows. The folder actually lists examples of each in `02_headings_list.html`.

### Block-level elements

A block element starts on a **new line** and stretches to take up the **full width** available. Blocks stack on top of each other, one below the next. Examples from the folder's own list:

```html
<h1>Block level elements</h1>
<ul>
    <li>heading h1 to h6</li>
    <li>ol, li</li>
    <li>li</li>
    <li>p</li>
</ul>
```

So `<h1>` to `<h6>`, `<ol>`, `<ul>`, `<li>`, `<p>`, and `<div>` are all block-level.

### Inline elements

An inline element does **not** start a new line. It sits inside the flow of text and takes up only as much width as its content needs. The folder lists these too:

```html
<h1>Inline elements</h1>
<ul>
    <li>a tags</li>
    <li>img</li>
    <li>strong</li>
    <li>em</li>
</ul>
```

So `<a>`, `<img>`, `<strong>`, and `<em>` are inline.

```
How the two flow on the page:

  BLOCK elements stack vertically, each on its own full-width row:

    ┌───────────────────────────────────────┐
    │  <h1> heading                         │
    ├───────────────────────────────────────┤
    │  <p> a paragraph of text              │
    ├───────────────────────────────────────┤
    │  <ul> a list                          │
    └───────────────────────────────────────┘

  INLINE elements sit side by side within a line of text:

    This is a <a>link</a> and some <strong>bold</strong> text
    ────────── ────────── ───────── ──────────────────── ─────
    all flowing together on the same line
```

### `<div>` and `<span>`: the generic containers

Two elements exist purely for grouping, with no meaning of their own. `<div>` is a block-level generic container, and `<span>` is an inline generic container. The folder uses `<div>` to group the two lists:

```html
<div>
    <h1>Block level elements</h1>
    <ul> ... </ul>
</div>
<div>
    <h1>Inline elements</h1>
    <ul> ... </ul>
</div>
```

Each `<div>` bundles a heading and a list together as one section, which is handy for styling or positioning the group with CSS later. Use `<div>` when you need a block wrapper and `<span>` when you need to wrap a few words inline, but prefer semantic elements (section 13) when a meaningful one fits.

---

## 11. Tables

**Source:** `03_tables.html`

Tables display data in **rows and columns**, like a spreadsheet. Use them for genuine tabular data (schedules, price lists, records), not for page layout. The building blocks are:

- **`<table>`** wraps the whole table.
- **`<tr>`** is a table row (a horizontal line of cells).
- **`<th>`** is a header cell (bold and centered by default).
- **`<td>`** is a normal data cell.

Here is the second table from `03_tables.html`, which has a proper header row:

```html
<table>
    <tr>
        <th>Name</th>
        <th>Email</th>
        <th>DOB</th>
    </tr>
    <tr>
        <td>Hitesh</td>
        <td>h@hitesh.com</td>
        <td>02/02/2024</td>
    </tr>
    <tr>
        <td>Mr. H</td>
        <td>h@hitesh.com</td>
        <td>02/02/2024</td>
    </tr>
</table>
```

This renders as:

```
  ┌─────────┬───────────────┬────────────┐
  │  Name   │  Email        │  DOB       │   ← <tr> of <th> (headers)
  ├─────────┼───────────────┼────────────┤
  │  Hitesh │  h@hitesh.com │ 02/02/2024 │   ← <tr> of <td> (data)
  ├─────────┼───────────────┼────────────┤
  │  Mr. H  │  h@hitesh.com │ 02/02/2024 │   ← <tr> of <td> (data)
  └─────────┴───────────────┴────────────┘
```

The key mental model: a table is built **row by row**. Each `<tr>` is one horizontal row, and inside it you place one cell (`<th>` or `<td>`) per column. The columns line up because every row has the same number of cells in the same order.

```
How rows and cells map to the grid:

  <table>
    <tr> → row 1   <th>Name</th> <th>Email</th> <th>DOB</th>
    <tr> → row 2   <td>Hitesh</td> <td>...</td> <td>...</td>
    <tr> → row 3   <td>Mr. H</td>  <td>...</td> <td>...</td>
  </table>
             col 1        col 2       col 3
```

The first table in the file uses only `<td>` cells with no `<th>`, so it has no bold headers. Using `<th>` for the top row (as the second table does) is better because it marks those cells as headers, which helps screen readers announce "the Name column" when reading data.

### Extra table structure (worth knowing)

For larger tables, HTML offers grouping tags that the folder does not use but that make tables clearer and easier to style:

- **`<thead>`** groups the header row(s).
- **`<tbody>`** groups the main data rows.
- **`<tfoot>`** groups a footer row (like totals).
- **`<caption>`** gives the table a title.
- The attributes **`colspan`** and **`rowspan`** let one cell stretch across multiple columns or rows.

```html
<table>
    <caption>User records</caption>
    <thead>
        <tr><th>Name</th><th>Email</th></tr>
    </thead>
    <tbody>
        <tr><td>Hitesh</td><td>h@hitesh.com</td></tr>
    </tbody>
</table>
```

---

## 12. Forms

**Source:** `04_forms.html`

Forms are how a web page **collects input** from the user: login details, search text, choices, and settings. Everything the user types or selects can be sent to a server for processing. The `<form>` element wraps a set of input controls.

Here is the login form from `04_forms.html`:

```html
<form>
    <label for="email">
        email:
        <input placeholder="test" type="text" name="email">
    </label>
    <label for="password">
        password:
        <input placeholder="enter your password" type="password" name="password" id="">
    </label>
    <button type="reset">reset</button>
</form>
```

### The `<input>` element

`<input>` is the workhorse of forms. It is an empty element, and its `type` attribute completely changes what it becomes:

| `type` value | What it creates |
|---|---|
| `text` | a single-line text box |
| `password` | a text box that hides characters as dots |
| `checkbox` | a small box you can tick |
| `color` | a color picker |
| `email` | a text box that checks for a valid email |
| `number` | a box that accepts only numbers |
| `date` | a date picker |
| `radio` | a round button where only one in a group can be chosen |
| `file` | a file upload button |
| `submit` | a button that sends the form |

Notice how the login form uses `type="text"` for the email and `type="password"` for the password, so the password shows as dots while you type.

Other useful input attributes seen here:

- **`placeholder`** shows faint hint text inside the box before the user types (like "enter your password"). It disappears once they start typing. It is a hint, not a real value.
- **`name`** is the label used when the data is sent to the server. Every input that should be submitted needs a `name`.

### The `<label>` element

A `<label>` describes what an input is for. It matters for two reasons: it tells users what to type, and clicking the label focuses or toggles the input, which makes checkboxes and radio buttons much easier to click.

The `for` attribute on a label should match the `id` of its input to connect them:

```html
<label for="email">email:</label>
<input type="text" name="email" id="email">
```

> **A note on the folder's code:** in `04_forms.html` the labels have `for="email"` and `for="password"`, but the matching inputs do not have an `id` set (one even has `id=""`, which is empty). So the `for` and `id` are not actually linked. The form still works because the inputs are placed *inside* the labels (which also connects them), but the cleanest approach is to give each input a real `id` that matches its label's `for`.

### The second form: more input types

```html
<form>
    <label for="Games">
        Basketball: <input type="checkbox" name="" id="">
    </label>
    <label for="Games">
        <select name="Games" id="">
            <option value="Cricket">Cricket</option>
            <option value="Baseball">Baseball</option>
        </select>
    </label>
    <label for="month">
        <input type="color" name="" id="">
    </label>
    <button type="submit">submit</button>
</form>
```

This form introduces two more controls:

- **`<select>` with `<option>`**: a dropdown menu. The `<select>` is the box, and each `<option>` is a choice inside it. The `value` attribute on each option is what gets sent to the server when that option is chosen.
- **`type="color"`**: opens a color picker so the user can choose a color visually.

### Buttons and the `type` attribute

The `<button>` element creates a clickable button. Its `type` decides what it does inside a form:

- **`type="submit"`** sends the form data (this is the default).
- **`type="reset"`** clears the form back to its starting values.
- **`type="button"`** does nothing on its own, used with JavaScript for custom actions.

```
The flow of a form:

  User fills inputs  ──►  clicks a submit button  ──►  browser gathers every
                                                       input's name + value
                                                            │
                                                            ▼
                                                   sends the data to a server
                                                   (the "action" URL) for
                                                   processing

  The login form here has no "action", so it would just
  reload the page. Real forms add action="..." and method="post".
```

Two form attributes the folder leaves out but that matter in real projects:

- **`action`** on the `<form>` is the URL where the data is sent.
- **`method`** is how it is sent: `get` puts data in the URL (for searches), `post` sends it in the request body (for logins and private data).

---

## 13. Semantic HTML

**Source:** `05_semantics.html`

**Semantic** HTML means using tags that describe the **meaning** of their content, not just tags that group things visually. Instead of wrapping everything in generic `<div>` boxes, you use elements whose names say what the content *is*: a header, a navigation menu, an article, a footer.

Why bother? Because meaning helps three audiences that cannot see your visual layout:

1. **Screen readers** can jump straight to the navigation or main article for blind users.
2. **Search engines** understand your page better and rank it more accurately.
3. **Other developers** (including future you) read the code and instantly see the structure.

Here is the full semantic layout from `05_semantics.html`:

```html
<body>
    <header>
      <hgroup>
        <h1>Learn to create websites</h1>
        <h2>You can learn with hitesh and build your own websites</h2>
      </hgroup>
      <nav>
        <ul>
          <li><a href="#">Home</a></li>
          <li><a href="#">About Us</a></li>
          <li><a href="#">Contact Us</a></li>
        </ul>
      </nav>
    </header>

    <article>
      <hgroup>
        <h1>Lorem ipsum dolor sit.</h1>
        <h2>Lorem ipsum dolor sit amet consectetur adipisicing.</h2>
        <h3><time datetime="2024-08-02">August 2nd</time></h3>
      </hgroup>

      <section>
        <p>Lorem ipsum dolor, sit amet consectetur adipisicing elit...</p>
      </section>

      <section>
        <code>console.log("hitesh")</code>
      </section>
    </article>

    <footer>
      <address>add your address here</address>
      <p>&copy; chaicode 2024</p>
    </footer>
</body>
```

Let us go through each semantic element used here:

- **`<header>`** is the top area of the page or a section. It usually holds the title and the main navigation. Here it wraps the site title and the nav menu.
- **`<nav>`** marks a block of navigation links. Notice the links are built as a `<ul>` of `<li>` items, which is the standard, accessible way to build a menu. Screen readers can announce "navigation" and let users skip it.
- **`<article>`** is a self-contained piece of content that would make sense on its own, like a blog post, a news story, or a product card. If you could copy it to another site and it still made sense, it is an article.
- **`<section>`** groups related content within an article or page, usually around a theme. This file uses two sections inside the article to separate the text from the code sample.
- **`<footer>`** is the bottom area, typically holding copyright, contact info, and secondary links.
- **`<hgroup>`** groups a heading together with its subheading so they are treated as one title unit.
- **`<address>`** marks contact information, like a physical address or email for the page owner.
- **`<time datetime="2024-08-02">`** marks a date or time. The human-readable text is "August 2nd", while the `datetime="2024-08-02"` attribute gives a machine-readable version that computers can parse and sort.

```
The semantic layout of a typical page:

  ┌─────────────────────────────────────────────┐
  │  <header>   title + <nav> menu               │
  ├─────────────────────────────────────────────┤
  │  <nav>  Home | About Us | Contact Us         │
  ├─────────────────────────────────────────────┤
  │  <article>                                   │
  │    <section>  main text                      │
  │    <section>  code sample                    │
  │  </article>                                  │
  ├─────────────────────────────────────────────┤
  │  <footer>   address + copyright              │
  └─────────────────────────────────────────────┘

  Compare with the non-semantic way, where all of these
  would just be <div> ... </div> with no meaning.
```

Other useful semantic elements not in this file: **`<main>`** (the one main content area of the page, there should be only one), **`<aside>`** (side content like a sidebar or related links), and **`<figure>`** with **`<figcaption>`** (covered next).

> **The golden rule:** reach for a meaningful element first. Use `<div>` only when no semantic element fits. A page built from `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, and `<footer>` is far clearer than a page built from a pile of `<div>` boxes.

---

## 14. Media: Figures, Video, and Audio

**Source:** `media.html`

Beyond plain images, HTML can embed captioned figures, video, and audio directly in the page. The `media.html` file shows figures and video.

### `<figure>` and `<figcaption>`

When an image needs a caption, wrap the image in a `<figure>` and add a `<figcaption>`. This groups the picture and its caption as one meaningful unit.

```html
<figure>
    <img
        src="images/hc.png"
        alt="The head and torso of a dinosaur skeleton;
             it has a large head with long sharp teeth"
        width="200"
    />
    <figcaption>
        A T-Rex on display in the Manchester University Museum.
    </figcaption>
</figure>
```

The difference between a caption in a `<figcaption>` and a plain `<p>` under an image is meaning: the browser and assistive tools know the `<figcaption>` describes the image it is grouped with. Notice this example also has a proper, descriptive `alt`, which is exactly the good practice from section 8.

### `<video>`

The `<video>` element embeds a video player. From `media.html`:

```html
<video
    src="./images/chai_pe_charcha-1080.mp4"
    width="400px"
    controls
    controlslist="noremoteplayback"
></video>
```

Key attributes:

- **`src`** is the path to the video file (here an `.mp4`).
- **`controls`** is a boolean attribute (it has no value, just its presence matters). It tells the browser to show the play, pause, volume, and timeline controls. Without it, the user has no way to play the video.
- **`controlslist`** fine-tunes those controls. `noremoteplayback` hides the "cast to another device" button.

Other common video attributes worth knowing: **`autoplay`** (starts on its own, usually needs `muted` to work), **`loop`** (repeats), **`muted`** (starts silent), and **`poster`** (an image to show before it plays).

```
Boolean attributes:

  Some attributes are just on/off switches. Their presence
  means "on", and you do not give them a value.

  <video controls>      → controls are ON
  <video>               → controls are OFF

  Other examples: autoplay, loop, muted, disabled, checked, required
```

### `<audio>` (worth knowing)

Audio works the same way as video, just without a picture:

```html
<audio src="song.mp3" controls></audio>
```

### Providing multiple sources

Not every browser supports every file format. The robust pattern is to offer several `<source>` files inside the media element, and the browser picks the first one it can play. Fallback text shows if none work:

```html
<video width="400" controls>
    <source src="clip.mp4" type="video/mp4">
    <source src="clip.webm" type="video/webm">
    Your browser does not support video.
</video>
```

---

## 15. HTML Entities

**Source:** `05_semantics.html`

Some characters cannot be typed directly into HTML because the browser would misread them. For example, `<` and `>` are used for tags, so if you want to *show* a less-than sign as text, you need another way. The answer is **HTML entities**: special codes that start with `&` and end with `;`.

The footer in `05_semantics.html` uses one:

```html
<p>&copy; chaicode 2024</p>
```

`&copy;` produces the copyright symbol ©. Here are the most common entities:

| Entity code | Shows | Why you need it |
|---|---|---|
| `&copy;` | © | copyright symbol |
| `&lt;` | < | the less-than sign (else read as a tag) |
| `&gt;` | > | the greater-than sign |
| `&amp;` | & | the ampersand itself |
| `&nbsp;` | (space) | a non-breaking space that will not collapse |
| `&quot;` | " | a double quote |
| `&mdash;` | — | a long dash |

```
Why entities exist:

  You type:      <p>5 &lt; 10 &amp; 10 &gt; 5</p>
  Browser shows: 5 < 10 & 10 > 5

  If you typed the raw < and > and &, the browser would
  try to read them as tags and get confused.
```

The `&nbsp;` entity is especially handy: normal spaces collapse (section 2), but a non-breaking space always stays, so you can force a gap or keep two words on the same line.

---

## 16. Good Habits and Common Mistakes

This section pulls together the best-practice notes scattered through the guide, several of which come straight from small issues in the folder's own files.

**Always write meaningful `alt` text.** `01_intro.html` uses `alt=""`, which tells screen readers to skip the image. Unless an image is purely decorative, describe it. `media.html` does this correctly.

**Link labels with their inputs.** In `04_forms.html` the `for` values do not match real `id` values on the inputs. Give each input an `id` and point the label's `for` to it. This makes forms accessible and lets users click the label to focus the field.

**Do not skip heading levels for size.** `01_intro.html` jumps from `<h2>` to `<h4>`. Keep headings in order and use CSS to change sizes.

**Use one `<h1>` per page.** It names the page's single main topic.

**Prefer semantic tags over `<div>`.** Reach for `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, and `<footer>` when they fit, as `05_semantics.html` does. Fall back to `<div>` only when nothing semantic applies.

**Close your tags and nest them properly.** Tags should close in the reverse order they opened. `<p><em>text</em></p>` is correct, `<p><em>text</p></em>` is not.

**Always include `charset`, `viewport`, and a `title`.** Every file in the folder does this. The viewport meta tag in particular is what makes pages usable on phones.

**Indent your code consistently.** It has no effect on the output, but clean indentation (as in `05_semantics.html` and `media.html`) makes the structure obvious and easy to maintain.

**Validate your HTML.** Free tools like the W3C Markup Validator catch unclosed tags, wrong nesting, and typos before they cause bugs.

---

## 17. Quick Reference — Mental Model

```
HTML — how everything in this folder connects:

  ┌────────────────────────────────────────────────────────────────────┐
  │  EVERY PAGE STARTS THE SAME WAY                                     │
  │  <!DOCTYPE html> → <html lang> → <head> (meta, title) → <body>     │
  └───────────────────────────────┬────────────────────────────────────┘
                                  │  everything visible lives in <body>
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │  TEXT CONTENT    │  │  GROUPING        │  │  MEDIA & LINKS       │
  │  h1-h6 headings  │  │  block vs inline │  │  <a href>  links     │
  │  <p> paragraphs  │  │  <div> <span>    │  │  <img>     images    │
  │  <em> <strong>   │  │  <ul> <ol> <li>  │  │  <figure>  captions  │
  │  <code>          │  │                  │  │  <video> <audio>     │
  └──────────────────┘  └──────────────────┘  └──────────────────────┘
            │                     │                     │
            ▼                     ▼                     ▼
  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
  │  TABLES          │  │  FORMS           │  │  SEMANTIC LAYOUT     │
  │  <table>         │  │  <form>          │  │  <header> <nav>      │
  │  <tr> row        │  │  <input type>    │  │  <main> <article>    │
  │  <th> header     │  │  <label>         │  │  <section> <aside>   │
  │  <td> data cell  │  │  <select>/option │  │  <footer>            │
  │                  │  │  <button>        │  │  (meaning, not divs) │
  └──────────────────┘  └──────────────────┘  └──────────────────────┘

  Threads running through all of it:
    • tags wrap content, attributes configure tags
    • whitespace collapses; only tags change layout
    • meaning first: choose the tag that describes what content IS
    • accessibility: alt text, labels, and semantics help everyone
```

### Cheat sheet of everything used in this folder

| Tag / attribute | File it appears in | What it does |
|---|---|---|
| `<!DOCTYPE html>` | all | declares modern HTML5 |
| `<html lang="en">` | all | root element, sets language |
| `<head>` / `<title>` | all | metadata / browser-tab text |
| `<meta charset>` / `<meta viewport>` | all | encoding / mobile scaling |
| `<h1>`–`<h6>` | intro, lists, semantics | headings, largest to smallest |
| `<p>` | intro, lists, semantics | paragraph |
| `<em>` / `<code>` | lists, semantics | emphasis / inline code |
| `<a href>` | intro, semantics | link (absolute or relative) |
| `<img src alt width>` | intro, lists, media | image |
| `<ol>` / `<ul>` / `<li>` | lists | ordered / unordered lists |
| `<div>` | lists | generic block container |
| `<table>` `<tr>` `<th>` `<td>` | tables | tabular data |
| `<form>` | forms | collects user input |
| `<input type>` | forms | text, password, checkbox, color |
| `<label for>` | forms | describes an input |
| `<select>` / `<option>` | forms | dropdown menu |
| `<button type>` | forms | submit / reset button |
| `<header>` `<nav>` `<footer>` | semantics | page structure regions |
| `<article>` / `<section>` | semantics | self-contained / thematic content |
| `<hgroup>` `<address>` `<time>` | semantics | heading group / contact / date |
| `<figure>` / `<figcaption>` | media | image with a caption |
| `<video controls>` | media | embedded video player |
| `title="..."` attribute | intro, media | hover tooltip |
| `&copy;` entity | semantics | copyright symbol © |

---

*All examples taken directly from the `html/` course files: `index.html`, `01_intro.html`, `02_headings_list.html`, `03_tables.html`, `04_forms.html`, `05_semantics.html`, and `media.html`. Extra concepts (main, aside, audio, multiple sources, table grouping, form action/method) were added where the folder leaves gaps.*
