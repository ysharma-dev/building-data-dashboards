# Chapter 15 — Environment Variables and Secrets

**Skill:** connecting any app to any external API safely — keeping secret
credentials out of your code and out of Git history — and building a
fallback path so your app still runs, and is still demoable, even without
live credentials. This is one of the highest-stakes skills in this whole
book: getting it wrong doesn't just cause a bug, it can leak a credential to
the entire internet.

## Why secrets can't just live in your code

To talk to the Harness API, our app needs an **API key** — a secret string
that proves to Harness's servers "this request is authorized, on behalf of
this account." Anyone who has that string can make requests as if they were
you.

It might seem convenient to just write it directly into a file:

```ts
// DO NOT DO THIS
const apiKey = "abc123-super-secret-key";
```

Here's why that's dangerous: recall from [Chapter 3](03-git-and-github-basics.md)
that Git keeps a full history of every version of every file you've ever
committed. If you commit that line and later realize your mistake and
delete it, **the old commit still has it** — anyone with access to the
repository's history (including, if you ever push it to a public GitHub
repository, literally anyone on the internet) can find it. Automated bots
constantly scan public GitHub repositories specifically looking for leaked
API keys, and a leaked key gets used within minutes of being found. Fully
scrubbing a secret from Git history after the fact is possible but painful
— far better to never commit it in the first place.

## Environment variables: the standard solution

The standard fix is an **environment variable**: a named value that lives
*outside* your code, in your local environment (or your hosting provider's
configuration, once deployed), and gets read by your code at runtime rather
than being written into it.

Next.js has a specific, built-in convention for this: a file named
`.env.local` at your project root. Create it now:

```bash
touch .env.local
```

Put your Harness configuration inside it, in `KEY=value` format, one per
line:

```
HARNESS_API_KEY=your-actual-secret-key-here
HARNESS_ACCOUNT_ID=your-account-id-here
HARNESS_BASE_URL=https://app.harness.io
```

(If you're using a different Harness instance than the default, your
account's own URL will be given to you when you generate your API key —
see the box below.)

Next.js automatically loads every variable in `.env.local` and makes it
available in your server-side code as `process.env.HARNESS_API_KEY`, and so
on. No import needed — it's just there.

### Getting a real Harness API key (optional)

If you want to use the app against real deployment data: create a free
[Harness](https://www.harness.io/) account, then in the Harness UI go to
your profile menu → **My API Keys** (or your account admin settings) and
generate a **Personal Access Token**. Copy it into `.env.local` as
`HARNESS_API_KEY` above, along with your account identifier (visible in
your account settings) as `HARNESS_ACCOUNT_ID`.

**You don't need to do this to follow the rest of the book.**
[Appendix A](appendix-a-mock-data-fallback.md) shows the mock-data fallback
we're setting up in this very chapter — the app runs and looks identical
either way.

## Making sure Git never sees it

`create-next-app` already generated a `.gitignore` file (introduced in
[Chapter 3](03-git-and-github-basics.md)) that includes a line like:

```
.env*
```

That pattern tells Git to ignore any file starting with `.env` — including
`.env.local`. Confirm this is actually working before you do anything else:

```bash
git status
```

`.env.local` should **not** appear in the output at all (not even as
"untracked"). If it does show up, double check your `.gitignore` file
contains that `.env*` line, add it if missing, and re-run `git status`.
This is worth verifying now, once, carefully — it's much cheaper than
discovering later that a secret got committed.

## The example file convention

There's a companion convention worth adopting: alongside the real,
git-ignored `.env.local`, commit a *second* file — `.env.local.example` —
that lists the same variable **names**, with blank or placeholder values:

```
HARNESS_API_KEY=
HARNESS_ACCOUNT_ID=
HARNESS_BASE_URL=
```

This file is safe to commit (no secrets in it) and serves as living
documentation: anyone who clones your project later (including future-you,
in six months) can see exactly which environment variables the app expects,
without needing to read through every file to find every `process.env.`
reference.

## Failing loudly when config is missing

Where should code that needs `HARNESS_API_KEY` actually go? We'll build the
real Harness client fully in [Chapter 17](17-talking-to-the-harness-api.md),
but the very first thing that file will do is read and validate this
configuration:

```ts
function getConfig() {
  const apiKey = process.env.HARNESS_API_KEY;
  const accountId = process.env.HARNESS_ACCOUNT_ID;
  const baseUrl = process.env.HARNESS_BASE_URL;

  if (!apiKey || !accountId || !baseUrl) {
    throw new Error(
      "Missing Harness config. Set HARNESS_API_KEY, HARNESS_ACCOUNT_ID, HARNESS_BASE_URL in .env.local",
    );
  }

  return { apiKey, accountId, baseUrl };
}
```

Notice this doesn't try to guess or silently fall back to some default —
it throws an immediate, specific error naming exactly which variables are
missing. This is a deliberate, general pattern: when required configuration
is absent, fail fast and loud, with a message that tells the *next* person
(often future-you) exactly what to fix, rather than letting the app limp
along and fail confusingly three steps later.

## The mock-data fallback

Now, the part that lets you follow this entire book without ever creating a
Harness account. We're going to add one more environment variable —
`USE_MOCK_DATA` — that, when set to `"true"`, tells our data-access layer
(built in Chapter 17) to return realistic made-up data instead of calling
the real Harness API at all.

Add this to `.env.local`:

```
USE_MOCK_DATA=true
```

And to `.env.local.example` too, so it's documented:

```
USE_MOCK_DATA=
```

In [Chapter 17](17-talking-to-the-harness-api.md), every function in our
Harness client will start with a tiny check like this:

```ts
function useMockData(): boolean {
  return process.env.USE_MOCK_DATA === "true";
}

export async function listOrgs(): Promise<HarnessOrg[]> {
  if (useMockData()) return mockOrgs();
  // ...real Harness API call goes here
}
```

This is a genuinely useful pattern beyond just this book: a small, explicit
"escape hatch" flag that swaps a real, slow, credential-requiring data
source for a fast, deterministic, fake one — useful not just for readers
without an account, but for local development, demos, and automated tests
in any real project. [Appendix A](appendix-a-mock-data-fallback.md) builds
out the full set of mock data this flag switches to.

## Checkpoint

- [ ] `.env.local` exists at your project root with `HARNESS_API_KEY`,
      `HARNESS_ACCOUNT_ID`, `HARNESS_BASE_URL`, and `USE_MOCK_DATA`.
- [ ] `git status` does **not** show `.env.local` anywhere in its output.
- [ ] `.env.local.example` exists, is safe to commit, and lists the same
      variable names with blank values.
- [ ] You can explain, in your own words, why committing a secret to Git
      is dangerous even if you delete it in a later commit.

**This generalizes to:** every project that talks to a paid or
authenticated external service — a payments API, a maps API, an email
service, anything — needs this exact same treatment: secrets in an
untracked `.env.local`-style file, a committed example file documenting
the shape, loud failure when configuration is missing, and (where practical)
a mock/fake mode so the project remains runnable and demoable without live
credentials in front of it.

Next: [Chapter 16 — Defining Types](16-defining-types.md)
