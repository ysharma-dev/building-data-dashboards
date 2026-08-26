---
layout: chapter
title: "Chapter 4 — JavaScript Crash Course"
nav_order: 5
permalink: /04-javascript-crash-course/
---

# Chapter 4 — JavaScript Crash Course

Every single piece of logic in the dashboard app you'll build — reading
data, filtering it, calculating a metric, deciding what to show on screen —
is written in JavaScript (or its close relative, TypeScript, which you'll
meet in [Chapter 5](05-typescript-crash-course.md)). This chapter teaches
you the specific pieces of JavaScript this book's project actually uses —
not the whole language, which is far bigger than what any one project
needs, but exactly the vocabulary you need to read and write the code in
later chapters. If a term shows up in this book's code and isn't covered
here, that's a bug in the book — but the goal was to cover everything.

JavaScript is a **programming language** — a precise, limited vocabulary
for giving a computer instructions. Unlike a human language, every sentence
(called a **statement**) has to follow exact rules, or the computer won't
understand it and will show you an error instead of guessing what you
meant.

## Variables: `let` and `const`

A **variable** is a named container that holds a value, so you can refer to
that value later by name instead of retyping it.

```js
let count = 0;
const name = "Ada";
```

Both lines create a variable and give it a starting value. The difference
is whether the value is allowed to change afterward:

- `let` creates a variable whose value **can** be reassigned later.
- `const` creates a variable whose value **cannot** be reassigned after it's
  set (short for "constant"). Trying to do so causes an error.

```js
let count = 0;
count = count + 1; // fine — count is now 1

const name = "Ada";
name = "Grace"; // error — name was declared with const
```

A common beginner question: if `const` can't change, why not just always
use `let`? In practice, most values in a well-written program never
actually need to change once set, and using `const` for those makes your
intent clear and lets the computer catch a mistake (accidentally reassigning
something you didn't mean to) for you. The convention this book follows —
and one you'll see throughout real JavaScript and TypeScript code — is:
default to `const`, and only use `let` when you specifically know a value
needs to be reassigned later (like a counter that increases in a loop).

## Basic types

Every value in JavaScript has a **type** — a category describing what kind
of thing it is and what you can do with it. The core types you'll see
throughout this book are:

**String** — text, always wrapped in quotes:

```js
const status = "Success";
```

**Number** — a numeric value (JavaScript doesn't distinguish whole numbers
from decimals the way some languages do — they're both just "number"):

```js
const durationInSeconds = 42;
const successRate = 0.73;
```

**Boolean** — one of exactly two values, `true` or `false`, used for
yes/no, on/off style facts:

```js
const isRunning = true;
const hasFailed = false;
```

**Array** — an ordered list of values, written inside square brackets and
separated by commas:

```js
const statuses = ["Success", "Failed", "Running"];
```

Arrays can hold any type of value, including other arrays or objects (next).
You access one item by its position, called its **index** — counting starts
at `0`, not `1`:

```js
statuses[0]; // "Success"
statuses[1]; // "Failed"
```

**Object** — a collection of named values, written inside curly braces as
`key: value` pairs separated by commas. This is how you'll represent a
single "thing" with multiple properties — for example, one deployment:

```js
const deployment = {
  id: "exec-123",
  status: "Success",
  durationInSeconds: 42,
};
```

You access a property on an object using a dot, followed by its name:

```js
deployment.status; // "Success"
deployment.durationInSeconds; // 42
```

Objects are the shape you'll use constantly in this project — a deployment,
a pipeline, a filter selection are all represented as objects with named
properties.

## Functions

A **function** is a named, reusable block of instructions. Instead of
writing the same steps over and over, you write them once inside a
function, then **call** (run) that function by name whenever you need those
steps done.

### Regular function syntax

```js
function double(number) {
  return number * 2;
}

double(5); // 10
```

Breaking this down:

- `function double(...)` names the function `double`.
- `number` inside the parentheses is a **parameter** — a placeholder for a
  value the function expects to be given when it's called.
- The code between `{` and `}` is the function's **body** — what it
  actually does.
- `return` sends a value back out of the function to whoever called it. A
  function that doesn't explicitly `return` anything gives back a special
  "nothing" value called `undefined`.
- `double(5)` **calls** the function, passing `5` in as the value for
  `number`. This is called an **argument** — the actual value supplied at
  call time, as opposed to `number`, the parameter name used inside the
  function's definition.

### Arrow function syntax

JavaScript has a second, shorter way to write functions, called an **arrow
function**, using `=>` (read as "goes to" or "arrow"). You'll see this style
far more often than the `function` keyword throughout this book's later
chapters, especially for small functions.

```js
const double = (number) => {
  return number * 2;
};

double(5); // 10
```

This does exactly the same thing as the version above — it's just a
different way of writing "a function that takes `number` and returns
`number * 2`." When a function's entire body is a single `return`
statement, arrow functions allow an even shorter form, dropping both the
curly braces and the word `return`:

```js
const double = (number) => number * 2;

double(5); // 10
```

You'll see this compact style constantly, especially inside the array
methods covered later in this chapter.

## Template literals

A **template literal** is a way of writing text (a string) using backticks
(`` ` ``) instead of regular quotes, which lets you embed variables and
expressions directly inside the text using `${...}`.

```js
const name = "Ada";
const count = 3;

const message = `${name} has ${count} deployments`;
// "Ada has 3 deployments"
```

Compare that to building the same string without template literals, gluing
pieces together with `+`:

```js
const message = name + " has " + count + " deployments";
```

Both produce the same result, but template literals are easier to read and
far less error-prone once you're combining several values — and they're
what you'll see used throughout this book's code whenever text needs to
include a variable's value.

## `async`/`await`: dealing with things that take time

Some operations don't finish instantly. The biggest example in this book:
fetching data from an API over the network (Chapter 17 does exactly this)
can take anywhere from a few milliseconds to a few seconds, depending on
network conditions completely outside your program's control.

Here's the problem this creates: if your program just stopped and waited,
frozen, every time it asked for data over the network, the whole app would
become unresponsive during that wait — no button clicks, no screen updates,
nothing — for however long the network happened to take. That's a bad
experience, and in a browser, a fully frozen page is exactly what
JavaScript is designed to avoid.

JavaScript's solution is to mark operations like "fetch this data" as
**asynchronous** — meaning "this will finish at some point in the future,
not necessarily right now," and give you a specific, structured way to say
"and here's what to do once it's actually done," without freezing
everything else while you wait.

The `async`/`await` pattern is the modern, readable way to write this.
First, a function that contains any waiting has to be marked `async`:

```js
async function getDeployments() {
  const response = await fetch("https://example.com/api/deployments");
  const data = await response.json();
  return data;
}
```

Two things to notice:

- The function is declared with `async function` instead of just
  `function`. This marks it as a function that may need to wait for
  something.
- `await` is placed in front of an operation that takes time (`fetch(...)`,
  which requests data over the network, and `response.json()`, which reads
  the response's contents). `await` means "pause *this function* here until
  this finishes, then continue with the result" — but crucially, it does
  **not** freeze the rest of your program while it waits. Everything else
  keeps running normally; only the code after this specific `await`, inside
  this specific function, waits for it.

You'll see this exact pattern — `async function`, with `await` in front of
anything that fetches data — throughout Part 3 of this book, especially
Chapter 17 (talking to the Harness API) and Chapter 18 (API route
handlers). For now, the key idea to hold onto is: **`async`/`await` exists
because some things take time, and JavaScript needs a clean way to say
"wait for this specific result" without freezing everything else.**

## Array methods used later in this book

Arrays come with built-in functions (called **methods** when they belong to
a specific value, like an array) for transforming and inspecting their
contents. These five show up constantly once you get to Chapter 22
(computing metrics from a list of deployments), so it's worth getting
comfortable with them now, using small, obvious examples.

All five take a function as an argument — a function you provide, describing
what to do with each item. Passing a function into another function like
this is an extremely common JavaScript pattern, and arrow functions (from
earlier in this chapter) are almost always used for it.

### `.map()` — transform every item into something new

Produces a **new array** the same length as the original, with each item
replaced by the result of running your function on it.

```js
const numbers = [1, 2, 3];
const doubled = numbers.map((n) => n * 2);
// doubled is [2, 4, 6]
```

Later in this book, you'll use `.map()` to turn a list of raw deployment
records into a list of just their statuses, or to transform data into the
exact shape a chart library expects.

### `.filter()` — keep only some items

Produces a **new array** containing only the items for which your function
returns `true`.

```js
const numbers = [1, 2, 3, 4, 5, 6];
const evens = numbers.filter((n) => n % 2 === 0);
// evens is [2, 4, 6]
```

(`%` is the **remainder operator** — `n % 2` gives the remainder after
dividing `n` by 2, which is `0` for even numbers.)

You'll use `.filter()` constantly for the dashboard's filter bar —
"keep only the deployments matching the selected status," for example.

### `.find()` — get the first matching item

Returns the **first single item** for which your function returns `true`
(not an array of matches — just one item, or `undefined` if nothing
matches).

```js
const deployments = [
  { id: "a", status: "Failed" },
  { id: "b", status: "Success" },
];

const failedOne = deployments.find((d) => d.status === "Failed");
// failedOne is { id: "a", status: "Failed" }
```

(`===` checks whether two values are equal. JavaScript also has `==`, which
is looser about matching different types — this book always uses `===`,
which is the safer, more predictable choice and the one you'll see used
throughout modern JavaScript code.)

### `.reduce()` — combine every item into a single result

The most flexible (and, for beginners, least intuitive) of the five. It
walks through the array, building up a single accumulated result as it
goes.

```js
const numbers = [1, 2, 3, 4];
const total = numbers.reduce((sum, n) => sum + n, 0);
// total is 10
```

Reading this carefully: `reduce` takes two arguments — a function, and a
starting value (`0` here). The function itself takes two parameters: `sum`
(the running total so far) and `n` (the current item). For each item in the
array, it computes `sum + n` and that becomes the new `sum` for the next
item. It starts with `sum = 0`, then becomes `1`, then `3`, then `6`, then
`10`.

This is exactly the shape you'll use to compute a total or an average
across a list of deployments in Chapter 22 — for example, adding up every
deployment's duration to compute an average.

### `.sort()` — put items in order

Reorders the array's items. Without any argument, `.sort()` converts items
to text and sorts alphabetically, which is rarely what you want for
numbers — so you'll almost always pass a function describing how to compare
two items:

```js
const numbers = [30, 5, 100, 1];

const ascending = numbers.sort((a, b) => a - b);
// ascending is [1, 5, 30, 100]
```

The comparison function's rule: if it returns a negative number, `a` is
placed before `b`; if positive, `a` is placed after `b`. `a - b` produces a
negative number whenever `a` is smaller — which is exactly the rule for
smallest-to-largest order. You'll use this in Chapter 24 to sort
deployments by date or duration for the deep-dive table.

> **Note:** unlike `.map()` and `.filter()`, `.sort()` changes the original
> array in place, rather than only returning a new one — a detail worth
> remembering if a later chapter's code looks like it's being careful to
> copy an array before sorting it.

## Checkpoint

- You can explain the difference between `let` and `const`, and know the
  default should be `const`.
- You can read an object like `{ id: "a", status: "Failed" }` and access its
  `status` property with a dot.
- You can explain, in plain language, why `async`/`await` exists — because
  some operations (like fetching data) take time, and freezing the whole
  program while waiting would be a bad experience.
- Given a small array, you could predict the output of `.map()`,
  `.filter()`, and `.reduce()` on it.

Next: [Chapter 5 — TypeScript Crash Course](05-typescript-crash-course.md)
