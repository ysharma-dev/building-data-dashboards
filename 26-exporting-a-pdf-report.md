# Chapter 26 — Exporting a PDF Report

**Skill:** turning any on-screen view into a shareable, printable artifact
— a PDF, in this case — including the two problems that show up in nearly
every real "export what's on screen" feature: waiting for asynchronous
content to actually finish loading before you capture it, and keeping the
exported file a reasonable size.

## The two libraries, and what each one actually does

Exporting "whatever is currently on screen" as a PDF takes two separate
steps, and two separate libraries:

1. **Turn a piece of the DOM into an image.** The DOM (Document Object
   Model — the browser's in-memory representation of everything on the
   page) isn't directly a PDF-able thing; we need to render it as a
   picture first. [`html2canvas-pro`](https://github.com/yorickshan/html2canvas-pro)
   does this — point it at any HTML element, and it draws a pixel-for-pixel
   snapshot of that element onto an in-memory `<canvas>` (a browser element
   built specifically for drawing raster graphics).
2. **Turn that image (or several) into a PDF file.** [`jsPDF`](https://github.com/parallax/jsPDF)
   builds an actual PDF document, page by page, and lets you place an
   image (or text, or shapes) onto each page.

Install both:

```bash
npm install html2canvas-pro jspdf
```

**A quick note on the name:** this project uses `html2canvas-pro`, a
maintained fork of the original `html2canvas`, specifically because the
original library couldn't correctly parse `oklch()` colors — recall
[Chapter 14](14-configuring-tailwind-and-shadcn.md), where every one of
our design tokens is defined using exactly that color format. If you ever
hit a rendering tool that mysteriously fails on colors that work fine
everywhere else, checking whether it supports whatever *specific* CSS
color syntax you're using is a reasonable first thing to check — this is
exactly what happened here during the app's real development.

## Dynamic imports: only load what you need, when you need it

Both libraries are reasonably large, and they're only ever needed the
moment someone actually clicks "Export." Loading them into every single
page visit, even for the vast majority of visitors who never click that
button, would be wasteful. Instead, we load them **dynamically** — only
when the export function actually runs:

```tsx
const [{ default: html2canvas }, { jsPDF }] = await Promise.all([
  import("html2canvas-pro"),
  import("jspdf"),
]);
```

`import(...)` used as a function call (rather than the `import ... from`
statement you've used everywhere else in this book) is JavaScript's
**dynamic import** — it returns a Promise that resolves once that module
has actually been downloaded and is ready to use. Wrapping both in
`Promise.all(...)` (recall [Chapter 4](04-javascript-crash-course.md) and
[Chapter 17](17-talking-to-the-harness-api.md)'s worker-pool pattern) lets
both libraries download simultaneously rather than one after the other.
This is a genuinely useful, broadly applicable technique: any feature that
needs a large library but is used by only a fraction of your visitors is a
good candidate for a dynamic import instead of a normal top-of-file one.

## Waiting for the page to actually be ready

Here's the first real problem, and a subtle one. The Option 1 deep dive
([Chapter 24](24-the-option1-deep-dive.md)) loads its data *asynchronously*
— when the page first renders, it briefly shows "Analyzing N child
deploys…" before the real content appears. If someone clicks "Export"
during that brief loading window, a naive capture would freeze that exact
loading message into the PDF forever — not the actual report they wanted.

There's no simple prop or event this button can check to know "is
everything on the page finished loading" — the loading state lives deep
inside a completely different component. Instead, we poll the *rendered
text itself*, watching for a specific pattern to disappear:

```tsx
const LOADING_TEXT_PATTERN = /Analyzing \d+ child deploys|Loading…/;

async function waitForContentToSettle(node: HTMLElement, timeoutMs = 20000): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    if (!LOADING_TEXT_PATTERN.test(node.innerText)) return;
    await new Promise((resolve) => setTimeout(resolve, 300));
  }
}
```

`node.innerText` reads the actual, currently-rendered visible text of
whatever DOM element `node` refers to — in this case, the whole
report section. The regular expression `/Analyzing \d+ child deploys|Loading…/`
(recall regex syntax from [Chapter 17](17-talking-to-the-harness-api.md))
matches either of the two loading messages this app can show. The loop
checks, roughly every 300 milliseconds, whether that pattern is *still*
present anywhere on the page — as soon as it's gone (meaning the real
content has replaced it), the function returns immediately, letting the
export proceed. A `timeoutMs` ceiling (20 seconds here) prevents this from
waiting forever if something's genuinely stuck; after that, the export
proceeds anyway rather than hanging the button indefinitely.

This is a real, if slightly unusual, technique worth having in your
toolkit: **when you have no formal signal for "is this finished," and the
thing you're waiting on always produces a specific, recognizable bit of
visible text while it's still working, you can poll for that text's
absence as a proxy for "done."** It's not the most elegant possible
solution (a proper shared loading-state signal, threaded through props,
would be cleaner in a codebase built for it from scratch), but it's a
pragmatic, working fix for exactly the situation this feature was added
into.

## Capturing and slicing into pages

Once the content has settled, the actual capture happens:

```tsx
const isDark = document.documentElement.classList.contains("dark");
const canvas = await html2canvas(node, {
  scale: 1.5,
  useCORS: true,
  backgroundColor: isDark ? "#0a0a0a" : "#ffffff",
});
```

`scale: 1.5` renders at 1.5x the element's normal pixel size — sharper
than a 1:1 capture, without going all the way to a full 2x, which (as
you'll see below) matters for file size. `backgroundColor` is set
explicitly, checked against whether dark mode (recall
[Chapter 14](14-configuring-tailwind-and-shadcn.md)'s `.dark` class) is
currently active — without this, a transparent background could render
unpredictably once placed onto a PDF page, which is always opaque.

A single dashboard, captured as one tall image, would be an awkward PDF —
either squeezed illegibly onto one giant page, or with no natural page
breaks at all. Instead, the tall canvas gets **sliced into several
page-sized chunks**:

```tsx
const pageWidth = canvas.width;
const pageHeight = Math.round(pageWidth * 1.414); // A4 aspect ratio
const totalPages = Math.max(Math.ceil(canvas.height / pageHeight), 1);

const pdf = new jsPDF({ orientation: "portrait", unit: "px", format: [pageWidth, pageHeight] });

for (let page = 0; page < totalPages; page++) {
  const sliceCanvas = document.createElement("canvas");
  sliceCanvas.width = pageWidth;
  sliceCanvas.height = pageHeight;
  const ctx = sliceCanvas.getContext("2d");
  if (!ctx) continue;

  ctx.fillStyle = isDark ? "#0a0a0a" : "#ffffff";
  ctx.fillRect(0, 0, pageWidth, pageHeight);
  ctx.drawImage(canvas, 0, -page * pageHeight);

  const imgData = sliceCanvas.toDataURL("image/jpeg", 0.85);
  if (page > 0) pdf.addPage([pageWidth, pageHeight]);
  pdf.addImage(imgData, "JPEG", 0, 0, pageWidth, pageHeight);
}
```

`1.414` is the ratio of an A4 page's height to its width — a standard
paper size — so each resulting PDF page has realistic, printable
proportions rather than an arbitrary shape. For each page, a *fresh*
blank canvas is created, filled with the background color, and then the
*original* full-height canvas is drawn onto it at a vertical offset
(`-page * pageHeight`) — effectively "sliding a window" down the tall
original image, one page-height at a time, and capturing whatever falls
inside that window onto its own page. `pdf.addPage(...)` adds a new blank
page to the PDF (skipped for the very first page, which the constructor
already created), and `pdf.addImage(...)` places that slice's image onto
it.

## Keeping the file size reasonable

The original version of this feature used PNG images (a lossless format —
no visual quality lost, but resulting in much larger files) and produced
roughly a 39-megabyte PDF for one dashboard export — unreasonably large
for something meant to be quickly shared over email or chat. Two changes
fixed this:

```tsx
const imgData = sliceCanvas.toDataURL("image/jpeg", 0.85);
```

Switching to **JPEG at 85% quality** (a *lossy* format — some visual
detail is discarded to shrink the file, controlled by that `0.85`
parameter) combined with the `scale: 1.5` capture setting from earlier
(rather than a sharper `2`) brought the same export down to roughly a few
megabytes. The reasoning: this is a dashboard full of text, charts, and
flat-colored UI — not a photograph — so the kind of fine-grained visual
detail JPEG compression tends to blur is barely present in the first
place, meaning the *visual* quality loss is negligible while the file-size
win is substantial. This is a genuinely broad lesson: whether lossy
compression is an acceptable tradeoff depends entirely on *what* you're
compressing — a photograph might show real artifacts at 85% JPEG quality
where a screenshot of text and UI barely will.

## Naming the file usefully

```tsx
function slugify(value: string): string {
  return value.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "");
}

const datePart = new Date().toISOString().slice(0, 10);
const namePart = [orgName, projectName, pipelineName]
  .filter((v): v is string => Boolean(v))
  .map(slugify)
  .join("-");
pdf.save(`harness-deploy-insights${namePart ? `-${namePart}` : ""}-${datePart}.pdf`);
```

`slugify` turns any human-readable string into something safe to use in a
filename — lowercase, with anything that isn't a letter, digit, or hyphen
replaced by a hyphen, and stray leading/trailing hyphens trimmed off.
`new Date().toISOString().slice(0, 10)` produces a `YYYY-MM-DD` date
string, a sortable, unambiguous date format. The `.filter(Boolean)`
(recall this pattern from earlier chapters) drops any of `orgName`,
`projectName`, `pipelineName` that happen to be missing, so the final
filename gracefully includes only whatever context is actually available
— something like `harness-deploy-insights-default-ieng-training-2026-08-24.pdf`.

## Tracking status for the button itself

The whole export flow takes real time (waiting for content, then actually
capturing and building the PDF), so the button needs its own small state
machine, again following the loading/error/data pattern from
[Chapter 21](21-fetching-and-displaying-executions.md), adapted slightly
for an action rather than a fetch:

```tsx
const [status, setStatus] = useState<"idle" | "waiting" | "exporting" | "error">("idle");
```

Four states instead of three, because there are genuinely two different
kinds of "in progress" worth showing distinctly to the person waiting:
`"waiting"` (for the content-settle check above) and `"exporting"` (the
actual capture-and-build work) — each with slightly different button text,
so the person clicking it has an accurate sense of what's currently
happening, rather than one generic "Loading…" the whole time.

## Checkpoint

- [ ] `npm install html2canvas-pro jspdf` has been run.
- [ ] Clicking export while data is still loading correctly waits, rather
      than capturing a frozen loading message.
- [ ] The resulting PDF has multiple pages, each roughly an A4 shape, and
      is a few megabytes rather than tens of megabytes.
- [ ] The downloaded filename includes today's date and, when available,
      your selected org/project/pipeline names.

**This generalizes to:** any "export what's on screen" feature needs the
same two-part treatment — a real, working way to know the content has
finished loading before you capture it (even if that means a pragmatic
workaround like text-polling, when no cleaner signal exists), and a
deliberate choice about image format and resolution that trades a
negligible amount of visual fidelity for a dramatically smaller, more
shareable file. And whenever a library import is only needed behind one
specific user action, a dynamic `import(...)` keeps that cost out of every
other page load.

**This is Piece #10 from the anatomy table** in
[Chapter 0](00-introduction.md) — Export.

Next: [Chapter 27 — Polish, Accessibility, and Bugfixes](27-polish-accessibility-and-bugfixes.md)
