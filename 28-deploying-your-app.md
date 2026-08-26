# Chapter 28 — Deploying Your App

**Skill:** taking any Next.js app from "runs on my machine" to a real,
shared URL other people can visit — including the one detail that trips up
almost every beginner the first time: environment variables have to be
configured *again*, separately, in your hosting provider, because your
`.env.local` file (recall [Chapter 15](15-environment-variables-and-secrets.md))
never leaves your own computer.

## Why "it works on my machine" isn't the finish line

Everything so far has run via `npm run dev` on your own computer, reachable
only at `localhost` — meaning only *you*, on this one machine, can see it.
Sharing an app with anyone else means **deploying** it: uploading your code
to a service that runs it continuously, on infrastructure that's reachable
from the public internet, at a real URL.

## Choosing a host: Vercel

[Vercel](https://vercel.com/) is the company that builds Next.js itself,
and its hosting platform is built specifically to deploy Next.js apps with
minimal configuration — a sensible default choice for this project. (Other
hosts — Netlify, Railway, AWS, and many more — can also run a Next.js app;
the general shape of what follows applies to most of them, even though the
exact clicks differ.)

## Step 1: push your code to GitHub

Recall [Chapter 3](03-git-and-github-basics.md): Vercel deploys by
connecting to a Git repository, so your project needs to be pushed to
GitHub (or a similar service) first.

```bash
git remote add origin git@github.com:your-username/harness-deploy-insights.git
git branch -M main
git push -u origin main
```

`git remote add origin <url>` tells your local repository about a remote
copy hosted elsewhere, naming it `origin` (a conventional default name —
you could call it anything, but `origin` is what nearly every tool and
tutorial expects). `git branch -M main` ensures your primary branch is
named `main` (the modern convention). `git push -u origin main` uploads
your commits to that remote, with `-u` recording the link between your
local `main` branch and `origin`'s `main` branch, so future pushes can
just be `git push` with no arguments.

**Before you push, double-check** (recall [Chapter 15](15-environment-variables-and-secrets.md)):
run `git status` one more time and confirm `.env.local` does not appear
anywhere in what's about to be pushed. This is the last checkpoint before
your code becomes visible outside your own machine (or to anyone else with
access to the repository, if it's shared) — worth the ten extra seconds.

## Step 2: connect the repository to Vercel

Sign up for a free Vercel account (you can sign in directly with your
GitHub account, which also grants Vercel permission to see your
repositories), then choose "Add New Project" and select your
`harness-deploy-insights` repository. Vercel automatically detects that
it's a Next.js project (from `package.json`'s dependencies) and pre-fills
the correct build settings — you shouldn't need to change anything there.

## Step 3: configure environment variables — again

This is the step almost every beginner misses the first time, and then
spends a confusing few minutes debugging: **your `.env.local` file is
git-ignored, on purpose (Chapter 15) — which means it was never uploaded
to GitHub, and Vercel has no way of knowing what's in it.** Your deployed
app will start up with `HARNESS_API_KEY` (and every other variable)
completely undefined, and immediately hit the loud "Missing Harness
config" error from [Chapter 15](15-environment-variables-and-secrets.md)
— which, in this specific case, is exactly the point: that error existing
at all is what makes this misconfiguration obvious and fast to diagnose,
rather than a mysterious silent failure.

In Vercel's project settings, find **Environment Variables**, and add each
one from your `.env.local.example` list, with real values this time:

```
HARNESS_API_KEY=<your-real-key>
HARNESS_ACCOUNT_ID=<your-real-account-id>
HARNESS_BASE_URL=<your-real-base-url>
USE_MOCK_DATA=<true, or leave unset>
```

This is a deliberate, secure design: your secret never has to be committed
to Git at any point, in any repository, public or private — it lives only
in your own `.env.local` on your computer, and separately, directly in
your hosting provider's own secrets storage. Two separate places holding
the same values, neither one derived from the other by copying a file.

## Step 4: deploy, and watch the build

Clicking "Deploy" tells Vercel to clone your repository, run
`npm install` (downloading every dependency listed in `package.json`,
recall [Chapter 13](13-project-setup.md)), then `npm run build` (compiling
your app for production — a different, optimized mode from
`npm run dev`'s development mode), and finally start serving the result.
Watch the build logs; if something fails here, the error message will
usually point directly at the problem — a missing environment variable, a
TypeScript error that somehow wasn't caught locally, or a dependency issue.

Once it succeeds, Vercel gives you a real, public URL — something like
`harness-deploy-insights.vercel.app` — that anyone, anywhere, can now
visit.

## What happens on every future push

Once connected, Vercel automatically watches your GitHub repository: every
time you `git push` to your main branch, Vercel automatically rebuilds and
redeploys your app with the new code — no manual redeploy step needed.
This is worth understanding conceptually even if you don't explore it
deeply here, because it's the standard shape of **continuous deployment**:
push code, and it goes live automatically, the same broad idea behind the
Harness CD pipelines this entire app is built to measure in the first
place ([Chapter 10](10-harness-and-cd-pipelines.md)) — a nice bit of
symmetry to notice as you finish this project.

## Checkpoint

- [ ] Your project is pushed to a GitHub repository, with `.env.local`
      confirmed absent from it.
- [ ] Your project is connected to Vercel (or your host of choice) and has
      successfully deployed.
- [ ] Every environment variable from `.env.local.example` has been
      re-entered, with real values, directly in your hosting provider's
      settings — not copied via any file.
- [ ] Visiting your new public URL from a different device (like your
      phone) shows the working app.

**This generalizes to:** whatever you build next, deploying it will
always require this same two-part realization: your code lives in Git and
gets rebuilt by your host automatically, but your *secrets* never travel
through Git at all — they have to be entered a second time, directly, into
whatever secrets-storage your hosting provider offers. Expect to hit this
exact "it works locally but the deployed version can't find its
configuration" moment at least once in your career, and now you'll
recognize it immediately instead of being stuck on it.

Next: [Chapter 29 — Building Your Own Dashboard](29-building-your-own-dashboard.md)
