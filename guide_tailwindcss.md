# Tailwind CSS — Zero to Hero

> A utility-first CSS framework for rapidly building custom designs without leaving your HTML.
> Current version: **4.x** | License: MIT

---

## Table of Contents

1. [What Is Tailwind CSS](#1-what-is-tailwind-css)
2. [Installation](#2-installation)
3. [Core Concept: Utility-First](#3-core-concept-utility-first)
4. [Responsive Design](#4-responsive-design)
5. [Dark Mode](#5-dark-mode)
6. [Theme Customization (@theme)](#6-theme-customization-theme)
7. [Colors](#7-colors)
8. [Typography](#8-typography)
9. [Layout (Flexbox & Grid)](#9-layout-flexbox--grid)
10. [Spacing, Sizing & Borders](#10-spacing-sizing--borders)
11. [States & Variants](#11-states--variants)
12. [Animations & Transitions](#12-animations--transitions)
13. [Container Queries](#13-container-queries)
14. [Custom Utilities & Plugins](#14-custom-utilities--plugins)
15. [Component Patterns](#15-component-patterns)
16. [Production Optimization](#16-production-optimization)

---

## 1. What Is Tailwind CSS

Tailwind CSS is a utility-first framework where you style elements by composing small, single-purpose classes directly in your markup. Instead of writing custom CSS, you use classes like `flex`, `pt-4`, `text-center`, and `bg-blue-500`.

**v4 is a ground-up rewrite** with a Rust-based Lightning CSS engine, CSS-first configuration, and zero-config content detection. Builds are 2-10x faster than v3.

---

## 2. Installation

### With Vite (recommended)

```bash
npm install tailwindcss @tailwindcss/vite
```

In `vite.config.ts`:

```typescript
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [tailwindcss()],
})
```

In your main CSS file (e.g. `src/style.css`):

```css
@import "tailwindcss";
```

That's it. No config file needed.

### With PostCSS (Webpack, etc.)

```bash
npm install tailwindcss @tailwindcss/postcss autoprefixer
```

In `postcss.config.js`:

```javascript
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
}
```

### With Next.js

```bash
npx create-next-app@latest my-app --tailwind
```

Or add to existing Next.js:

```bash
npm install tailwindcss @tailwindcss/postcss
```

### Standalone CLI (no JS build tool)

```bash
npx @tailwindcss/cli -i input.css -o output.css --watch
```

---

## 3. Core Concept: Utility-First

### Traditional CSS

```html
<div class="card">
  <h2 class="card-title">Hello</h2>
</div>
```

```css
.card { padding: 1.5rem; border-radius: 0.5rem; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
.card-title { font-size: 1.25rem; font-weight: 700; }
```

### Tailwind way

```html
<div class="p-6 rounded-lg shadow">
  <h2 class="text-xl font-bold">Hello</h2>
</div>
```

No custom CSS written. Every utility maps to exactly one CSS property.

---

## 4. Responsive Design

Tailwind uses **mobile-first** breakpoints as variant prefixes:

| Prefix | Min-width | Target |
|--------|-----------|--------|
| (none) | 0px | Mobile (default) |
| `sm:` | 640px | Small tablets |
| `md:` | 768px | Tablets |
| `lg:` | 1024px | Laptops |
| `xl:` | 1280px | Desktops |
| `2xl:` | 1536px | Large screens |

```html
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  <!-- 1 column on mobile, 2 on tablet, 3 on desktop -->
</div>
```

### Custom breakpoints

```css
@theme {
  --breakpoint-3xl: 1920px;
  --breakpoint-xs: 475px;
}
```

Now `3xl:` and `xs:` variants are available.

---

## 5. Dark Mode

### Automatic (v4 default)

By default, Tailwind v4 uses `prefers-color-scheme`:

```html
<div class="bg-white dark:bg-gray-900 text-black dark:text-white">
  Adapts to OS preference automatically.
</div>
```

### Manual toggle (class-based)

In your CSS:

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

Then toggle the `dark` class on `<html>`:

```javascript
document.documentElement.classList.toggle('dark')
```

---

## 6. Theme Customization (@theme)

v4 replaces `tailwind.config.js` with CSS-first `@theme` directives. Every variable you define automatically generates matching utility classes.

```css
@import "tailwindcss";

@theme {
  --color-brand-50: #eff6ff;
  --color-brand-500: #3b82f6;
  --color-brand-900: #1e3a8a;

  --font-family-display: "Inter", sans-serif;
  --font-family-mono: "Fira Code", monospace;

  --spacing-18: 4.5rem;
  --spacing-128: 32rem;

  --radius-pill: 9999px;

  --shadow-soft: 0 2px 8px rgba(0, 0, 0, 0.08);
}
```

Use in HTML:

```html
<h1 class="font-display text-brand-500">Branded heading</h1>
<div class="p-18 rounded-pill shadow-soft">Custom card</div>
```

### Removing default theme values

```css
@theme {
  --color-*: initial;             /* remove all default colors */
  --color-primary: #3b82f6;      /* add only what you need */
}
```

---

## 7. Colors

Tailwind ships with a modernized **P3 color palette** in v4.

### Background, text, border

```html
<div class="bg-blue-500 text-white border border-blue-700">
  Blue card
</div>
```

### Opacity modifier

```html
<div class="bg-black/50">50% transparent black</div>
<div class="text-white/80">80% white text</div>
```

### Gradients

```html
<div class="bg-linear-to-r from-blue-500 via-purple-500 to-pink-500">
  Gradient background
</div>
```

> Note: v4 renamed `bg-gradient-to-*` to `bg-linear-to-*`.

### Arbitrary colors

```html
<div class="bg-[#1da1f2] text-[rgb(255,255,255)]">
  Custom hex / rgb value
</div>
```

---

## 8. Typography

### Font size

```html
<p class="text-xs">0.75rem</p>
<p class="text-sm">0.875rem</p>
<p class="text-base">1rem</p>
<p class="text-lg">1.125rem</p>
<p class="text-xl">1.25rem</p>
<p class="text-2xl">1.5rem</p>
<p class="text-4xl">2.25rem</p>
```

### Font weight, style, alignment

```html
<p class="font-bold italic text-center tracking-wide leading-relaxed">
  Styled text
</p>
```

### Text decoration & transform

```html
<a class="underline decoration-2 decoration-blue-500 underline-offset-4">Link</a>
<span class="uppercase">shouting</span>
<span class="truncate">Very long text that gets cut off...</span>
```

### Prose (long-form content)

Install the typography plugin:

```bash
npm install @tailwindcss/typography
```

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
```

```html
<article class="prose lg:prose-xl">
  <h1>Article title</h1>
  <p>Rendered with beautiful default styles.</p>
</article>
```

---

## 9. Layout (Flexbox & Grid)

### Flexbox

```html
<div class="flex items-center justify-between gap-4">
  <div>Left</div>
  <div>Right</div>
</div>

<div class="flex flex-col items-start gap-2">
  <span>Stacked</span>
  <span>Items</span>
</div>
```

### Grid

```html
<div class="grid grid-cols-3 gap-6">
  <div>1</div>
  <div class="col-span-2">Spans 2 columns</div>
</div>

<div class="grid grid-cols-[200px_1fr_200px] gap-4">
  <aside>Sidebar</aside>
  <main>Content</main>
  <aside>Sidebar</aside>
</div>
```

### Centering (the classic problem)

```html
<div class="flex items-center justify-center h-screen">
  Perfectly centered.
</div>

<!-- or with grid -->
<div class="grid place-items-center h-screen">
  Also centered.
</div>
```

---

## 10. Spacing, Sizing & Borders

### Spacing

```html
<div class="p-4 px-6 py-2">Padding</div>
<div class="m-4 mx-auto my-8">Margin</div>
<div class="space-y-4">           <!-- Adds gap between children -->
  <p>Item 1</p>
  <p>Item 2</p>
</div>
```

### Sizing

```html
<div class="w-full max-w-md h-64 min-h-screen">
  Width and height utilities
</div>
```

### Borders & Rings

```html
<div class="border border-gray-300 rounded-lg">Bordered</div>
<div class="border-2 border-dashed border-red-500">Dashed red</div>
<input class="ring-2 ring-blue-500 ring-offset-2 rounded" />
```

---

## 11. States & Variants

### Pseudo-classes

```html
<button class="bg-blue-500 hover:bg-blue-700 active:bg-blue-800 focus:ring-2">
  Interactive button
</button>

<input class="border focus:border-blue-500 focus:outline-none disabled:opacity-50" />
```

### Group and peer

```html
<div class="group p-4 hover:bg-gray-100">
  <h3 class="group-hover:text-blue-600">Reacts to parent hover</h3>
</div>

<input class="peer" placeholder="Email" />
<p class="hidden peer-invalid:block text-red-500">Invalid email</p>
```

### First, last, odd, even

```html
<ul>
  <li class="first:pt-0 last:pb-0 odd:bg-gray-50 even:bg-white">Item</li>
</ul>
```

### Composable variants (v4)

```html
<div class="group-has-focus:opacity-100">
  Shows when any child of group has focus.
</div>
```

---

## 12. Animations & Transitions

### Transitions

```html
<button class="transition-colors duration-200 ease-in-out hover:bg-blue-700">
  Smooth color change
</button>

<div class="transition-all duration-300 hover:scale-105 hover:shadow-lg">
  Scales up on hover
</div>
```

### Built-in animations

```html
<div class="animate-spin">Spinner</div>
<div class="animate-ping">Ping effect</div>
<div class="animate-pulse">Pulsing skeleton</div>
<div class="animate-bounce">Bouncing arrow</div>
```

### Custom keyframes

```css
@theme {
  --animate-slide-in: slide-in 0.3s ease-out;
}

@keyframes slide-in {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}
```

```html
<div class="animate-slide-in">Slides in from left</div>
```

---

## 13. Container Queries

v4 has built-in container query support — no plugin needed.

```html
<div class="@container">
  <div class="@sm:flex @lg:grid @lg:grid-cols-2">
    Responds to container width, not viewport.
  </div>
</div>
```

| Variant | Container min-width |
|---------|-------------------|
| `@xs:` | 320px |
| `@sm:` | 384px |
| `@md:` | 448px |
| `@lg:` | 512px |
| `@xl:` | 576px |

---

## 14. Custom Utilities & Plugins

### Arbitrary values

```html
<div class="top-[117px] bg-[#1da1f2] grid-cols-[fit-content(200px)_1fr]">
  Any CSS value in square brackets.
</div>
```

### Arbitrary properties

```html
<div class="[mask-image:linear-gradient(to_bottom,black,transparent)]">
  Any CSS property via bracket syntax.
</div>
```

### Writing a plugin

```css
@import "tailwindcss";
@plugin "@tailwindcss/forms";
@plugin "@tailwindcss/typography";
@plugin "@tailwindcss/container-queries";
```

### Official plugins

| Plugin | Purpose |
|--------|---------|
| `@tailwindcss/forms` | Better default form styles |
| `@tailwindcss/typography` | Prose styling for rendered markdown/HTML |
| `@tailwindcss/container-queries` | Additional container query utilities |
| `@tailwindcss/aspect-ratio` | Aspect ratio utilities (legacy, now built-in) |

---

## 15. Component Patterns

### Card

```html
<div class="bg-white dark:bg-gray-800 rounded-xl shadow-md overflow-hidden max-w-sm">
  <img class="w-full h-48 object-cover" src="/photo.jpg" alt="" />
  <div class="p-6">
    <h3 class="text-lg font-semibold text-gray-900 dark:text-white">Title</h3>
    <p class="mt-2 text-gray-600 dark:text-gray-300">Description text goes here.</p>
    <button class="mt-4 px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition-colors">
      Action
    </button>
  </div>
</div>
```

### Navbar

```html
<nav class="flex items-center justify-between px-6 py-4 bg-white dark:bg-gray-900 shadow">
  <a class="text-xl font-bold text-gray-900 dark:text-white" href="/">Logo</a>
  <div class="flex items-center gap-6">
    <a class="text-gray-600 hover:text-gray-900 dark:text-gray-300 dark:hover:text-white" href="/about">About</a>
    <a class="text-gray-600 hover:text-gray-900 dark:text-gray-300 dark:hover:text-white" href="/contact">Contact</a>
    <button class="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600">Sign in</button>
  </div>
</nav>
```

### Badge

```html
<span class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
  Active
</span>
```

---

## 16. Production Optimization

### Automatic purging

v4 detects which classes you use and only generates those — no manual `content` configuration needed. Typical production CSS: **6-25 KB gzipped**.

### Minification

Handled automatically by the build tool (Vite, PostCSS CLI, etc.).

### Class sorting

Install the Prettier plugin for consistent class ordering:

```bash
npm install -D prettier-plugin-tailwindcss
```

### Performance tips

- Use `@theme` to remove unused default tokens (e.g. `--color-*: initial`)
- Avoid `@apply` in hot paths — it adds indirection
- Prefer utility classes directly in templates over extracted CSS

---

**Last updated:** March 2026
