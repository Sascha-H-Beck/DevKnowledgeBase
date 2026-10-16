# Vite — Zero to Hero

> Next-generation frontend build tool. Lightning-fast dev server, instant HMR, optimized production builds.
> Current version: **7.x** | License: MIT

---

## Table of Contents

1. [What Is Vite](#1-what-is-vite)
2. [Installation](#2-installation)
3. [Project Structure](#3-project-structure)
4. [Dev Server & HMR](#4-dev-server--hmr)
5. [Configuration (vite.config.ts)](#5-configuration-viteconfigts)
6. [Static Assets & Imports](#6-static-assets--imports)
7. [Environment Variables](#7-environment-variables)
8. [CSS & Preprocessors](#8-css--preprocessors)
9. [Plugins](#9-plugins)
10. [Proxy & API Forwarding](#10-proxy--api-forwarding)
11. [Server-Side Rendering (SSR)](#11-server-side-rendering-ssr)
12. [Build & Optimization](#12-build--optimization)
13. [Library Mode](#13-library-mode)
14. [Multi-Page Apps](#14-multi-page-apps)
15. [Recipes & Patterns](#15-recipes--patterns)

---

## 1. What Is Vite

Vite (French for "fast") solves two problems that plague traditional bundlers:

1. **Slow dev server start** — Vite serves source files over native ESM. The browser does the bundling. No full rebuild on startup.
2. **Slow HMR** — Only the changed module is replaced, regardless of app size. Updates land in the browser in single-digit milliseconds.

For production, Vite uses Rollup under the hood for tree-shaking, code-splitting, and optimized output.

---

## 2. Installation

### New project (with framework)

```bash
npm create vite@latest my-app
```

The interactive prompt lets you choose:

| Template | Description |
|----------|-------------|
| `vanilla` | Plain JS/TS, no framework |
| `vue` / `vue-ts` | Vue 3 |
| `react` / `react-ts` | React |
| `preact` / `preact-ts` | Preact |
| `svelte` / `svelte-ts` | Svelte |
| `lit` / `lit-ts` | Lit Web Components |

Then install and run:

```bash
cd my-app
npm install
npm run dev
```

### Add Vite to an existing project

```bash
npm install -D vite
```

Add scripts to `package.json`:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

Create a minimal `index.html` at the project root (Vite uses it as the entry point):

```html
<!DOCTYPE html>
<html lang="en">
<head><meta charset="UTF-8" /></head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

---

## 3. Project Structure

```
my-app/
├── public/              # Served as-is at / (no processing)
├── src/
│   ├── assets/          # Processed by Vite (hashed filenames)
│   ├── components/
│   ├── main.ts          # Entry point
│   └── vite-env.d.ts    # Type declarations for Vite
├── index.html           # Root HTML — Vite's entry
├── vite.config.ts       # Configuration
├── tsconfig.json
└── package.json
```

**Key rule:** `index.html` lives in the project root, not inside `public/`.

---

## 4. Dev Server & HMR

Start with:

```bash
npm run dev
```

Default: `http://localhost:5173`.

### How HMR works

- Vite watches file changes via native OS events.
- Only the changed module (and its direct dependents) are invalidated.
- The browser receives a WebSocket message and hot-swaps the module.
- **Framework plugins** (e.g. `@vitejs/plugin-vue`, `@vitejs/plugin-react`) make HMR preserve component state.

No configuration needed — it works out of the box.

---

## 5. Configuration (vite.config.ts)

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  root: '.',
  base: '/',
  publicDir: 'public',
  resolve: {
    alias: {
      '@': '/src',
    },
  },
  server: {
    port: 3000,
    open: true,
    cors: true,
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    minify: 'esbuild',
  },
})
```

### Dynamic config based on mode

```typescript
export default defineConfig(({ command, mode }) => {
  if (command === 'serve') {
    return { /* dev-only config */ }
  } else {
    return { /* build config */ }
  }
})
```

### Async config

```typescript
export default defineConfig(async ({ command }) => {
  const data = await loadSomething()
  return { /* config */ }
})
```

---

## 6. Static Assets & Imports

### Importing assets

```typescript
import imgUrl from './image.png'         // returns resolved URL
import rawText from './file.txt?raw'     // returns string content
import workerUrl from './worker.js?worker' // Web Worker
```

### The `public/` directory

Files in `public/` are served at root and never processed:

```html
<!-- public/favicon.ico → /favicon.ico -->
<link rel="icon" href="/favicon.ico" />
```

### JSON

```typescript
import data from './data.json'            // parsed object
import { field } from './data.json'       // tree-shakeable named import
```

---

## 7. Environment Variables

Vite exposes env variables on `import.meta.env`. Only variables prefixed with `VITE_` are exposed to client code.

### .env files

```
# .env
VITE_API_URL=https://api.example.com
VITE_APP_TITLE=My App
DB_PASSWORD=secret          # NOT exposed (no VITE_ prefix)
```

### Usage

```typescript
console.log(import.meta.env.VITE_API_URL)
console.log(import.meta.env.MODE)          // 'development' | 'production'
console.log(import.meta.env.DEV)           // true in dev
console.log(import.meta.env.PROD)          // true in production
```

### File priority (highest to lowest)

```
.env.production.local
.env.production
.env.local
.env
```

---

## 8. CSS & Preprocessors

### Built-in support

Vite handles `.css` imports, CSS Modules, and PostCSS out of the box.

```typescript
import './style.css'                       // global CSS
import styles from './Button.module.css'   // CSS Modules
```

### Preprocessors

Install the preprocessor — no Vite plugin needed:

```bash
npm install -D sass         # .scss / .sass
npm install -D less         # .less
npm install -D stylus       # .styl
```

### PostCSS

Create `postcss.config.js` and Vite picks it up automatically:

```javascript
export default {
  plugins: {
    autoprefixer: {},
  },
}
```

---

## 9. Plugins

### Official plugins

| Plugin | Install | Purpose |
|--------|---------|---------|
| `@vitejs/plugin-vue` | `npm i -D @vitejs/plugin-vue` | Vue 3 SFC support |
| `@vitejs/plugin-react` | `npm i -D @vitejs/plugin-react` | React Fast Refresh |
| `@vitejs/plugin-legacy` | `npm i -D @vitejs/plugin-legacy` | Legacy browser support |

### Using plugins

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import legacy from '@vitejs/plugin-legacy'

export default defineConfig({
  plugins: [
    vue(),
    legacy({ targets: ['defaults', 'not IE 11'] }),
  ],
})
```

### Plugin ordering

Use `enforce` to control execution order:

```typescript
{
  name: 'my-plugin',
  enforce: 'pre',    // before Vite core plugins
  // or 'post'       // after Vite build plugins
}
```

### Conditional plugins (dev only / build only)

```typescript
{
  name: 'dev-only-plugin',
  apply: 'serve',   // only during dev server
}
```

### Finding plugins

- npm: search `vite-plugin`
- Community list: [awesome-vite](https://github.com/vitejs/awesome-vite#plugins)

---

## 10. Proxy & API Forwarding

Avoid CORS issues in development by proxying API requests:

```typescript
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:4000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/socket.io': {
        target: 'ws://localhost:4000',
        ws: true,
      },
    },
  },
})
```

Now `fetch('/api/users')` in the browser hits `http://localhost:4000/users`.

---

## 11. Server-Side Rendering (SSR)

Vite has first-class SSR support via the Environment API (v6+).

### Entry files

```
src/
├── entry-client.ts    # mounts the app in the browser
└── entry-server.ts    # renders the app to HTML string
```

### Dev server integration

```typescript
import { createServer } from 'vite'

const vite = await createServer({
  server: { middlewareMode: true },
  appType: 'custom',
})

app.use(vite.middlewares)

app.use('*', async (req, res) => {
  const { render } = await vite.ssrLoadModule('/src/entry-server.ts')
  const html = await render(req.url)
  res.end(html)
})
```

### Production SSR build

```bash
vite build --ssr src/entry-server.ts
vite build                                # client bundle
```

---

## 12. Build & Optimization

### Production build

```bash
npm run build
```

Output in `dist/`. Includes:

- Tree-shaking (dead code removal)
- Code splitting (dynamic imports → separate chunks)
- CSS extraction and minification
- Asset hashing for cache busting

### Build options

```typescript
export default defineConfig({
  build: {
    target: 'esnext',
    outDir: 'dist',
    assetsInlineLimit: 4096,    // inline assets < 4 KB as base64
    cssCodeSplit: true,
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
        },
      },
    },
  },
})
```

### Preview

```bash
npm run preview    # serves dist/ locally on :4173
```

---

## 13. Library Mode

Build a reusable library instead of an app:

```typescript
export default defineConfig({
  build: {
    lib: {
      entry: 'src/index.ts',
      name: 'MyLib',
      fileName: (format) => `my-lib.${format}.js`,
    },
    rollupOptions: {
      external: ['vue'],
      output: {
        globals: { vue: 'Vue' },
      },
    },
  },
})
```

Produces ESM and UMD bundles ready for npm publishing.

---

## 14. Multi-Page Apps

```typescript
export default defineConfig({
  build: {
    rollupOptions: {
      input: {
        main: 'index.html',
        admin: 'admin/index.html',
      },
    },
  },
})
```

Each HTML file is a separate entry point with its own dependency graph.

---

## 15. Recipes & Patterns

### Glob imports

```typescript
const modules = import.meta.glob('./modules/*.ts')
// { './modules/a.ts': () => import('./modules/a.ts'), ... }

const eagerModules = import.meta.glob('./modules/*.ts', { eager: true })
```

### Web Workers

```typescript
const worker = new Worker(new URL('./worker.ts', import.meta.url), {
  type: 'module',
})
```

### WASM

```typescript
import init from './module.wasm?init'
const instance = await init()
```

### Testing

Vite pairs naturally with **Vitest** — same config, same plugins, same transforms:

```bash
npm install -D vitest
npx vitest
```

---

**Last updated:** March 2026
