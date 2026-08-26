# Chapter 14 — Configuring Tailwind and shadcn

**Skill:** setting up a reusable design-token and component system at the
very start of a project, so every screen you build afterward pulls from one
consistent set of colors, spacing, and components instead of you inventing
new one-off styles every time. Doing this early is what makes a fast-moving
project still look coherent after fifty components.

## What we're setting up, and why now

`create-next-app` already installed Tailwind CSS for us (we said "Yes" to
that question in [Chapter 13](13-project-setup.md)), and gave us a mostly
empty `app/globals.css`. Before we write our first real component, we're
going to do two things:

1. Define a full set of **design tokens** — named colors, radii, and fonts —
   so that later, when we write `bg-card` or `text-muted-foreground`, those
   names mean something specific and consistent, in both light and dark
   mode.
2. Install **shadcn/ui**, the tool that will generate our actual UI
   components (buttons, cards, dropdowns, etc.) styled against those
   tokens.

Doing this now, before any feature code exists, means every component we
add from here on automatically looks consistent — we're building the
palette before painting the picture.

## Tailwind v4's CSS-based configuration

If you've seen a Tailwind tutorial before, you may expect a
`tailwind.config.js` file. **This project doesn't have one.** Tailwind v4
(released after most existing tutorials were written) moved configuration
directly into your CSS file, using a special `@theme` block that Tailwind's
build tool understands.

Open `app/globals.css`. `create-next-app` already put a line like this near
the top:

```css
@import "tailwindcss";
```

That one line pulls in all of Tailwind's utility classes (`flex`,
`items-center`, `gap-2`, and hundreds more) — this is the "utility-first
CSS" idea from [Chapter 8](08-what-is-tailwind.md) in action.

Now we're going to add our own design tokens below it. First, two more
imports:

```css
@import "tailwindcss";
@import "tw-animate-css";
@import "shadcn/tailwind.css";
```

`tw-animate-css` adds a set of ready-made animation utility classes (for
things like a smooth accordion expand/collapse, used later in the book).
`shadcn/tailwind.css` is a small stylesheet that ships as part of the
`shadcn` package (installed below) with some baseline styles shadcn's
components expect. Install the packages this needs now, since we'll
reference them immediately:

```bash
npm install tw-animate-css
```

(The `shadcn` package itself gets installed automatically in a moment when
we run its setup command.)

## Declaring the dark-mode switch

Next, one line that defines *how* dark mode gets turned on:

```css
@custom-variant dark (&:is(.dark *));
```

Read this as: "treat any element that is a descendant of something with the
class `dark` as being in dark mode." In practice, this means: put the class
`dark` on your page's `<html>` element, and every Tailwind class prefixed
with `dark:` (like `dark:bg-black`) activates. Toggle that one class, and
your whole app's color scheme flips. You'll see exactly how that toggle
gets triggered automatically (based on the visitor's operating-system
preference) when we build `app/layout.tsx` in a later chapter.

## Mapping CSS variables to Tailwind tokens

Now the `@theme inline` block — this is the actual configuration Tailwind
v4 reads to know what your custom utility classes should mean:

```css
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-border: var(--border);
  /* ...and more, following the same pattern */

  --radius-sm: calc(var(--radius) * 0.6);
  --radius-md: calc(var(--radius) * 0.8);
  --radius-lg: var(--radius);
}
```

Notice the pattern: each line maps a Tailwind-facing name
(`--color-background`) to a plain CSS variable (`--background`). Why the
extra layer of indirection instead of just using `--background` directly?
Because it lets you define the *actual color values* separately, per theme
(light vs. dark), while the Tailwind-facing names never change. A
component that uses `bg-background` doesn't need to know or care whether
it's currently light or dark mode — it just always means "whatever the
background color currently is."

The radius lines follow the same idea for corner-rounding: one base
`--radius` variable, with several named sizes (`sm`, `md`, `lg`, and more)
computed from it as simple multiples. Change the one `--radius` value, and
every rounded corner across your entire app scales together.

## Defining the actual color values, per theme

Below the `@theme` block, you define what those plain CSS variables
(`--background`, `--card`, etc.) actually equal — once for light mode
(inside `:root`), and again for dark mode (inside a `.dark` class, which is
exactly what our `@custom-variant dark` rule above is watching for):

```css
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0 0);
  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);
  --border: oklch(0.922 0 0);
  --radius: 0.625rem;
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
  --card: oklch(0.205 0 0);
  --card-foreground: oklch(0.985 0 0);
  --primary: oklch(0.922 0 0);
  --primary-foreground: oklch(0.205 0 0);
  --muted: oklch(0.269 0 0);
  --muted-foreground: oklch(0.708 0 0);
  --border: oklch(1 0 0 / 10%);
}
```

Two things worth pausing on:

- **`oklch(...)` is just a way of writing a color**, similar in spirit to
  the `rgb(255, 0, 0)` you may have seen before, but using a different
  color model (Lightness, Chroma, Hue) that's better at producing colors
  that *look* evenly spaced to the human eye as you adjust one number at a
  time. You don't need to calculate these by hand — a design tool or
  shadcn's own theme generator produces them for you. What matters is the
  *pattern*: `:root` defines light-mode values, `.dark` overrides them with
  dark-mode values, using the exact same variable names both times.
- **The values genuinely flip.** `--background` is nearly white
  (`oklch(1 0 0)`) in light mode and nearly black (`oklch(0.145 0 0)`) in
  dark mode; `--foreground` (text color) does the opposite. Every component
  that uses `bg-background text-foreground` gets correct contrast in both
  modes automatically, with zero extra code in the component itself.

Finally, a small base-styles layer, applied once globally:

```css
@layer base {
  * {
    @apply border-border outline-ring/50;
  }
  body {
    @apply bg-background text-foreground;
  }
  html {
    @apply font-sans;
  }
}
```

`@apply` lets you use Tailwind utility classes *inside* a plain CSS rule —
here, giving every element a default border color and outline color, and
giving the page body its background/text colors and default font, all
sourced from the tokens you just defined.

## Installing shadcn/ui

With our design tokens in place, we can now install shadcn/ui — recall
from [Chapter 9](09-what-is-shadcn-ui.md): it's a CLI tool, not a
dependency you `import` from. Run its init command:

```bash
npx shadcn@latest init
```

It asks a few setup questions — base color (choose **Neutral**, a
gray-based palette; this is what the project's actual token values above
are drawn from), and whether to use CSS variables for theming (**Yes** —
that's exactly the `--background`/`--card`/etc. pattern we just built by
hand above; the init command can also write this for you if you're
starting completely fresh).

This generates a `components.json` file at your project root — shadcn's own
configuration, recording the choices you made so every future
`npx shadcn add <component>` command knows where to put files and which
conventions to follow:

```json
{
  "style": "base-nova",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "iconLibrary": "lucide",
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  }
}
```

A few fields worth understanding:

- `"tailwind": { "config": "" }` — deliberately empty, confirming there's no
  separate Tailwind config file to point at (Tailwind v4's CSS-based
  config, as we just built above).
- `"rsc": true` — shadcn should generate components compatible with React
  Server Components (the Server/Client Component distinction from
  [Chapter 7](07-what-is-nextjs.md)).
- `"iconLibrary": "lucide"` — shadcn will use the
  [Lucide](https://lucide.dev/) icon set for anything that needs an icon
  (a chevron, a checkmark, and so on).
- `"aliases"` — mirrors the `@/*` import alias from `tsconfig.json`,
  telling the CLI exactly which folder each category of generated file
  belongs in (UI primitives go in `components/ui/`, your own composed
  components in `components/`, shared logic in `lib/`).

## Installing your first two components

Now let's actually generate something. Run:

```bash
npx shadcn@latest add button card
```

This downloads the source code for a `Button` and a `Card` component and
writes them directly into `components/ui/button.tsx` and
`components/ui/card.tsx` in your project. Open one of them — `button.tsx`,
say. You'll see real, readable TypeScript and JSX, using Tailwind classes
and referencing the tokens you just defined (`bg-primary`,
`text-primary-foreground`, and so on). This is the "you own the code" idea
from Chapter 9 made concrete: nothing is hidden behind a package import you
can't inspect.

You can now use it anywhere in your app:

```tsx
import { Button } from "@/components/ui/button";

<Button>Click me</Button>
```

## Checkpoint

- [ ] `app/globals.css` has the `@import` lines, the `@custom-variant dark`
      line, a `@theme inline` block, and both a `:root` and a `.dark` block
      defining color variables.
- [ ] `npx shadcn@latest init` completed and created a `components.json`
      file at your project root.
- [ ] `npx shadcn@latest add button card` created
      `components/ui/button.tsx` and `components/ui/card.tsx`.
- [ ] You can open `button.tsx` and point to at least one Tailwind class in
      it that references a token you defined above (e.g. `bg-primary`).

**This generalizes to:** any project you build from here on — a support
dashboard, a personal blog, a SaaS product — benefits from the exact same
two-step setup: define your color/spacing tokens once, at the start, in one
place; then generate or hand-write components that reference those tokens
by name rather than hardcoding colors. Change the tokens later, and every
component that used them updates automatically. This is the difference
between a project that stays visually consistent as it grows and one that
slowly drifts into a mismatched patchwork of colors.

Next: [Chapter 15 — Environment Variables and Secrets](15-environment-variables-and-secrets.md)
