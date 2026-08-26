# Chapter 13 — Project Setup

**Skill:** scaffolding a new modern web app project — using an official tool
to generate a correct, working starting structure instead of assembling
config files by hand. This is the very first move of nearly every real-world
web project, regardless of what that project ends up being about.

## Why you don't build this by hand

A modern web app needs a lot of small pieces of configuration working
together correctly before you write a single feature: a build tool, a
TypeScript configuration, a folder structure the framework expects, linting
rules, and more. Getting all of that exactly right by hand, as a beginner,
would be tedious and error-prone. Instead, frameworks ship an official
"create" tool that generates a correct, minimal starting point for you —
you answer a few questions, and it writes the files.

For Next.js, that tool is `create-next-app`. Let's use it.

## Running create-next-app

Open your terminal (see [Chapter 1](01-terminal-basics.md) if you need a
refresher), navigate to the folder where you keep your projects, and run:

```bash
npx create-next-app@latest harness-deploy-insights
```

A quick note on `npx`: it's a tool that comes with npm (installed back in
[Chapter 2](02-installing-tools.md)) that downloads and runs a package
one time, without permanently installing it on your computer. You use `npx`
for tools you only need to run occasionally — like scaffolding a new
project — rather than tools your project depends on every time it runs.

You'll be asked a series of questions. Here's what each one means and which
answer to give:

```
Would you like to use TypeScript?          → Yes
Would you like to use ESLint?              → Yes
Would you like to use Tailwind CSS?        → Yes
Would you like your code inside a `src/` directory? → No
Would you like to use App Router?          → Yes
Would you like to use Turbopack?           → Yes (the default)
Would you like to customize the import alias (@/*)? → No (keep the default)
```

Let's unpack a few of these, since "just say yes" isn't really learning:

- **TypeScript** — Yes. We covered why in [Chapter 5](05-typescript-crash-course.md):
  it catches shape-mismatch mistakes before you ever run the code.
- **ESLint** — Yes. ESLint is a *linter*: a tool that reads your code and
  flags patterns that are likely mistakes or inconsistent style, without
  actually running the code. Think of it as an automated code reviewer that
  never gets tired of checking the same rules over and over.
- **Tailwind CSS** — Yes. We'll go deeper on this in
  [Chapter 8](08-what-is-tailwind.md) and [Chapter 14](14-configuring-tailwind-and-shadcn.md).
- **`src/` directory** — No. Some projects like to nest all their code
  inside a `src/` folder to separate it from config files at the project
  root. This project keeps `app/`, `components/`, and `lib/` directly at the
  root — a completely valid, common alternative. Either way works; we're
  matching the real project's actual layout.
- **App Router** — Yes, always, for this book. Next.js has an older way of
  organizing pages (the "Pages Router," using a `pages/` folder) that you
  may still see in older tutorials. This book only uses the newer **App
  Router** (an `app/` folder), which we introduced conceptually in
  [Chapter 7](07-what-is-nextjs.md).
- **Turbopack** — Yes, the current default. Turbopack is the tool that
  actually compiles and bundles your code as you develop — you don't need
  to understand its internals, just know it's what makes `npm run dev`
  (used below) fast.
- **Import alias** — Keep the default, `@/*`. This lets you write
  `import { cn } from "@/lib/utils"` instead of a fragile relative path like
  `../../lib/utils` — the `@/` always means "from the project root,"
  regardless of how deeply nested the file doing the importing is.

Once you answer everything, `create-next-app` will download some packages
and write a batch of files into a new `harness-deploy-insights` folder. This
can take a minute or two — you're waiting on npm to download packages over
the network.

## Touring the generated file tree

Move into the new folder and look around:

```bash
cd harness-deploy-insights
ls
```

You'll see something like this:

```
harness-deploy-insights/
├── app/
│   ├── favicon.ico
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── public/
│   ├── file.svg
│   ├── globe.svg
│   ├── next.svg
│   ├── vercel.svg
│   └── window.svg
├── next.config.ts
├── package.json
├── package-lock.json
├── postcss.config.mjs
├── eslint.config.mjs
├── tsconfig.json
└── node_modules/
```

Here's what each piece is for:

- **`app/`** — Your application code lives here. Recall from
  [Chapter 7](07-what-is-nextjs.md): a file's *path* inside `app/` becomes
  its URL. Right now there's just one page.
  - **`app/layout.tsx`** — The root layout. Every page in your app renders
    *inside* whatever this file returns — it's the shared shell (think:
    `<html>`, `<body>`, and anything that should appear on every single
    page, like navigation or, later in this project, a shared tooltip
    provider).
  - **`app/page.tsx`** — The actual home page component, rendered at the
    root URL (`/`).
  - **`app/globals.css`** — Global styles, imported once by the layout.
    This is where Tailwind gets wired in — more in
    [Chapter 14](14-configuring-tailwind-and-shadcn.md).
  - **`app/favicon.ico`** — The little icon shown in a browser tab.
- **`public/`** — Static files served as-is, directly by URL. A file at
  `public/globe.svg` is reachable at `yoursite.com/globe.svg`. The default
  project ships a few sample icons here; we won't use them.
- **`package.json`** — The project's manifest: its name, its dependencies
  (other people's code your project relies on, downloaded via npm — you
  saw this concept in [Chapter 2](02-installing-tools.md)), and its
  **scripts** — shortcut commands you can run with `npm run <script-name>`.
  Open it and look at the `"scripts"` section; you'll see `dev`, `build`,
  `start`, and `lint`.
- **`package-lock.json`** — An exact, automatically-generated record of
  every package version actually installed (including packages your
  dependencies themselves depend on). You never edit this by hand; npm
  manages it. Its job is to make sure that if someone else installs your
  project's dependencies later, they get the *exact* same versions you had,
  not just "roughly compatible" ones.
- **`node_modules/`** — Where all downloaded packages actually live on
  disk. It's usually huge (thousands of files) and is never committed to
  Git — [Chapter 19's](19-remaining-api-routes.md) sibling chapter on Git,
  [Chapter 3](03-git-and-github-basics.md), already set up a `.gitignore`
  file that excludes it, and `create-next-app` also generates one.
- **`next.config.ts`** — Next.js's own configuration file, for options that
  affect how the framework builds and runs your app. It starts nearly
  empty; you'll add one option to it in [Chapter 22](22-computing-metrics.md)'s
  sibling area of the project (the real project enables something called
  the "React Compiler" here — you'll see exactly what and why later).
- **`postcss.config.mjs`** — Configuration for PostCSS, a tool that
  processes your CSS as part of the build. You won't hand-edit this; it's
  what makes Tailwind's CSS-based configuration (Chapter 14) actually work
  during the build.
- **`eslint.config.mjs`** — ESLint's configuration, in the newer "flat
  config" format. You won't need to touch this either — the generated
  default is exactly what a Next.js + TypeScript project needs.
- **`tsconfig.json`** — TypeScript's configuration: which JavaScript
  features to target, how strict type-checking should be, and — important
  for later — the `@/*` import alias you agreed to when answering the
  setup questions.

## Running it for the first time

From inside the project folder, start the development server:

```bash
npm run dev
```

This runs the `dev` script defined in `package.json`, which starts
Next.js's local development server. After a moment you'll see output
telling you the app is ready, along with a local URL — typically
`http://localhost:3000`.

Open that URL in your web browser. `localhost` is a special hostname that
always means "this same computer" — you're not reaching out to the
internet at all; the app is running as a process on your machine, and your
browser is talking to it over your machine's own network stack. The `:3000`
is a *port number* — think of your computer as having thousands of
numbered doors it can listen on; `3000` happens to be Next.js's
conventional default door for local development.

You should see the default Next.js welcome page. That confirms everything
is wired up correctly: Node.js is installed, npm downloaded the packages
correctly, and Next.js's build tool can compile and serve your code.

Leave that terminal window running (it will keep printing logs and
auto-reload the page whenever you save a file), and open a **new** terminal
tab or window for the commands in the next chapter. This is a habit you'll
keep for the whole book: one terminal running `npm run dev` in the
background, another free for you to type commands into.

## Checkpoint

- [ ] `npx create-next-app@latest harness-deploy-insights` completed without
      errors, and you answered the setup questions as shown above.
- [ ] `cd harness-deploy-insights && npm run dev` starts a local server and
      prints a `localhost` URL.
- [ ] Opening that URL in your browser shows the default Next.js welcome
      page.
- [ ] You can identify, in your own words, what `app/page.tsx`,
      `package.json`, and `node_modules/` are each for.

**This generalizes to:** every framework you'll ever use — React, Vue,
Django, Ruby on Rails, whatever comes next — ships (or has a community
convention for) an official scaffolding tool just like `create-next-app`.
The specific questions differ, but the move is always the same: run the
official generator, answer honestly based on what your project actually
needs, then spend five minutes touring the resulting file tree before
writing anything, so you know what's already there before you add to it.

Next: [Chapter 14 — Configuring Tailwind and shadcn](14-configuring-tailwind-and-shadcn.md)
