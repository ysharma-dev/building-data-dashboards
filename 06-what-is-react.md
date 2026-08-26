---
layout: chapter
title: "Chapter 6 — What Is React?"
nav_order: 7
permalink: /06-what-is-react/
---

# Chapter 6 — What Is React?

Every visible piece of the dashboard you'll build in this book — the filter
dropdowns, the metric cards, the charts, the export button — is written
using **React**, a library for building user interfaces. Next.js (which
you'll meet in the next chapter) is built directly on top of React, so
before Next.js will make any sense, React itself needs to. This chapter
introduces React's core ideas using a tiny example you can hold entirely in
your head — none of this exact code reappears later, but every idea in it
does, over and over, starting in Chapter 13.

## What is a UI component?

A **component** is a reusable, self-contained piece of user interface —
something you build once and can then use repeatedly, each time
potentially showing different data.

Think about a button. Buttons appear all over a real app — "Export as
PDF," "Apply Filters," "Retry" — and they mostly look and behave the same
way, just with different text and a different action when clicked. Instead
of writing the HTML and behavior for a button from scratch every single
place one appears, you write it **once**, as a component, and then reuse it
wherever a button is needed, changing only what's different each time (its
label, what happens when it's clicked).

This is React's central idea: build your interface as a tree of small,
focused components, each responsible for one piece of the screen, and
combine them to build the whole page. The dashboard app in this book is,
under the hood, a few dozen components — a `FilterBar` component, a
`MetricCard` component, a `Chart` component, and so on — combined together.

## JSX: HTML-like syntax inside JavaScript

React components describe what should appear on screen using **JSX** — a
syntax that looks like HTML (if you've seen any web page source code
before) but is actually written directly inside JavaScript code.

```jsx
function Greeting() {
  return <h1>Hello, dashboard!</h1>;
}
```

That `<h1>Hello, dashboard!</h1>` is JSX. It looks like an HTML tag, but
it's not a string of text — it's a special syntax that gets converted (by
the same kind of build tooling mentioned in Chapter 5) into regular
JavaScript instructions that build up the actual page content.

Why does JSX exist, instead of just writing HTML and JavaScript
separately, the traditional way? Because a component's *appearance* and the
*data and logic that drive it* are usually deeply related — what a
component displays often depends directly on values computed in that same
component. JSX lets you weave the two together in one place: you can drop
any JavaScript value or expression directly into your markup by wrapping it
in curly braces `{}`.

```jsx
function Greeting() {
  const name = "Ada";
  return <h1>Hello, {name}!</h1>;
}
```

This renders as `Hello, Ada!` — the `{name}` is replaced with whatever
`name` currently holds. This is the same idea as the template literals
(`` `Hello, ${name}!` ``) from Chapter 4, just used inside markup instead of
a plain string.

## Props: data passed into a component

A component becomes genuinely reusable once it can behave differently based
on data given to it from outside. That data is called **props** (short for
"properties") — values passed into a component, similar to how arguments
are passed into a function (which, from Chapter 4, is exactly what a
component actually is under the hood: a function that returns JSX).

```jsx
function Greeting(props) {
  return <h1>Hello, {props.name}!</h1>;
}
```

You use a component, passing it props, using an HTML-attribute-like syntax:

```jsx
<Greeting name="Ada" />
<Greeting name="Grace" />
```

The first renders `Hello, Ada!`, the second `Hello, Grace!` — same
component, same code, different result, because of the different prop
passed in each time. This is exactly the mechanism you'll use in Chapter 21
onward to render one `MetricCard` component four times with four different
metrics, or one row of a table component once per deployment in a list.

## State: data a component remembers, via `useState`

Props are given to a component from outside and a component can't change
them itself. But components often need their own data that *can* change
over time, in response to something the user does — a dropdown's current
selection, whether a section is expanded, text typed into a search box.
This kind of data is called **state**, and in React, you create it using a
special function called `useState`.

Here is a small, complete, working example — a counter button — using every
idea from this chapter so far:

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Clicked {count} times
    </button>
  );
}
```

Walking through this line by line:

- `useState(0)` creates one piece of state, with a starting value of `0`.
- It returns **two things** together: the current value (`count`), and a
  function for changing it (`setCount`). The `const [count, setCount] =
  ...` syntax is called **array destructuring** — it's just a compact way
  of pulling the two values `useState` returns into two separate named
  variables in one line.
- `onClick={() => setCount(count + 1)}` attaches a function to the button's
  click event (note the arrow function syntax from Chapter 4). Every time
  the button is clicked, this function runs, calling `setCount` with the
  new value.

Here's the crucial mental model, and it's worth reading twice: **calling
`setCount` doesn't just change a variable somewhere — it tells React "this
component's data has changed, please re-run this component's function and
update the screen to match."** React components aren't like a traditional
program that runs once from top to bottom and stops. Instead, think of a
component's function as something React calls over and over, once for the
initial screen, and then again every single time its state changes — each
time producing the JSX describing what the screen should look like *right
now*, given the current state. React compares the new result to what's
already on screen and updates only what actually changed.

So in the counter example: click the button, `setCount(count + 1)` runs,
React re-runs the `Counter` function with the new `count` value, and the
text on screen updates from "Clicked 0 times" to "Clicked 1 times" —
automatically, with no code anywhere that manually finds the text on the
page and edits it. You describe *what the screen should show given the
current state*, and React handles making the screen match that description,
every time the state changes.

This exact pattern — a piece of state, a function to update it, and JSX
that reads from it — is what drives the dashboard's filter dropdowns
(Chapter 20), the expandable rows in its table (Chapter 24), and dozens of
other pieces of interactivity throughout Part 3.

## `useEffect`: running code in response to changes

`useState` handles data a component remembers. `useEffect` handles a
related but different need: running some code *in response to* a component
first appearing on screen, or one of its values changing — rather than in
response to a direct user action like a click.

```jsx
import { useState, useEffect } from "react";

function DeploymentCount() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("The count is now:", count);
  }, [count]);

  return <p>Deployments: {count}</p>;
}
```

`useEffect` takes two things: a function to run (here, one that logs the
current count), and an array of values to watch, called the **dependency
array** — here, `[count]`. React runs the effect function once when the
component first appears on screen, and then again every time any value in
the dependency array changes between renders. An empty array, `[]`, means
"only run this once, when the component first appears" — a pattern you'll
see constantly in Chapter 17 onward for fetching data as soon as a
component loads, using the `async`/`await` pattern from Chapter 4.

The general shape you'll see repeatedly starting in Part 3 looks like this:

```jsx
useEffect(() => {
  async function loadData() {
    const response = await fetch("/api/deployments");
    const data = await response.json();
    setDeployments(data);
  }

  loadData();
}, []);
```

Read this as: "as soon as this component appears on screen, go fetch the
deployment data, and once it arrives, store it in state" — which, per the
mental model above, causes the component to automatically re-render and
display the newly loaded data.

## Putting it together: the mental model

Hold onto this one sentence — it's the single most important idea in this
chapter, and everything in Chapters 20 through 26 builds directly on it:

**A React component is a function that returns JSX describing the screen,
and React re-runs that function automatically every time the component's
state changes, updating the screen to match.**

Props feed data in from outside. State holds data a component owns and can
change, usually in response to user interaction. `useEffect` runs code in
response to the component appearing or specific values changing, most often
used for fetching data. Everything else in React — and everything the
dashboard app does on screen — is built from these same few ideas, combined
and nested.

## Checkpoint

- You can explain, in your own words, what a component is and why
  reusability matters.
- You can read a small piece of JSX with a `{}` expression inside it and
  say what it renders.
- You can explain the difference between props (given from outside, can't
  be changed by the component) and state (owned by the component, can
  change).
- You can explain why calling `setCount(...)` updates what's on screen,
  using the "component re-runs its function" mental model.

Next: [Chapter 7 — What Is Next.js?](07-what-is-nextjs.md)
