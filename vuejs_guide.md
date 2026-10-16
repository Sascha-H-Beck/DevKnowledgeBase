# Vue.js — Zero to Hero: Complete Installation & Stack Guide

> A comprehensive, step-by-step guide covering Vue.js setup across every major stack — from standalone SPA to full-stack backends and desktop apps.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Stack A — Standalone Vue + Vite (Recommended Default)](#2-stack-a--standalone-vue--vite-recommended-default)
3. [Stack B — Nuxt 3 (Full-Featured Vue Framework)](#3-stack-b--nuxt-3-full-featured-vue-framework)
4. [Stack C — Vue + Laravel (Inertia.js)](#4-stack-c--vue--laravel-inertiajs)
5. [Stack D — Vue + Django REST Framework](#5-stack-d--vue--django-rest-framework)
6. [Stack E — Vue + Phoenix / Elixir (LiveVue)](#6-stack-e--vue--phoenix--elixir-livevue)
7. [Stack F — Vue + Electron (Desktop)](#7-stack-f--vue--electron-desktop)
8. [Stack G — Vue + Tauri 2 (Lightweight Desktop)](#8-stack-g--vue--tauri-2-lightweight-desktop)
9. [Core Concepts Cheat Sheet](#9-core-concepts-cheat-sheet)
10. [Recommended Extensions & Tooling](#10-recommended-extensions--tooling)

---

## 1. Prerequisites

Before starting with any stack, ensure you have the following installed.

### Node.js

Required version: **20.x or newer** (even-numbered LTS releases recommended: 20, 22).

Verify installation:

```bash
node --version
npm --version
```

If not installed, download from [nodejs.org](https://nodejs.org/) or use a version manager:

```bash
# Windows (winget)
winget install OpenJS.NodeJS.LTS

# macOS (Homebrew)
brew install node@22

# Linux (nvm — recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash
nvm install 22
nvm use 22
```

### Package Manager (pick one)

| Manager | Install | Speed | Disk usage |
|---------|---------|-------|------------|
| **npm** | Ships with Node.js | Baseline | Higher |
| **pnpm** | `npm install -g pnpm` | Fastest | Lowest (hard-linked store) |
| **yarn** | `corepack enable && yarn set version stable` | Fast | Medium |
| **bun** | `curl -fsSL https://bun.sh/install \| bash` | Very fast | Low |

This guide uses `npm` commands. Replace with your preferred manager where needed.

### Git

```bash
git --version
# If missing: https://git-scm.com/downloads
```

---

## 2. Stack A — Standalone Vue + Vite (Recommended Default)

The official, fastest way to start a Vue.js project. Uses `create-vue` (v3.22.0+) powered by Vite.

### Step 1 — Scaffold the project

```bash
npm create vue@latest
```

> **PowerShell users:** Quote dashes if passing flags:
> `npm create vue@latest '--' --typescript`

The interactive prompt asks you to configure:

| Prompt | Recommended for most projects |
|--------|-------------------------------|
| Project name | `my-app` |
| TypeScript | Yes |
| JSX Support | No |
| Vue Router | Yes |
| Pinia (state management) | Yes |
| Vitest (unit testing) | Yes |
| End-to-End Testing | Playwright |
| ESLint | Yes |
| Prettier | Yes |

### Step 2 — Install dependencies

```bash
cd my-app
npm install
```

### Step 3 — Start the dev server

```bash
npm run dev
```

Open `http://localhost:5173` in your browser. Hot Module Replacement (HMR) is active — changes appear instantly.

### Step 4 — Build for production

```bash
npm run build
```

Output lands in `dist/`. Serve it with any static host (Netlify, Vercel, Nginx, etc.).

### Step 5 — Preview the production build locally

```bash
npm run preview
```

### Project structure

```
my-app/
├── public/              # Static assets (favicon, etc.)
├── src/
│   ├── assets/          # Processed assets (images, CSS)
│   ├── components/      # Reusable Vue components
│   ├── router/          # Vue Router configuration
│   ├── stores/          # Pinia state stores
│   ├── views/           # Page-level components
│   ├── App.vue          # Root component
│   └── main.ts          # Entry point
├── index.html           # HTML shell
├── vite.config.ts       # Vite configuration
├── tsconfig.json        # TypeScript config
└── package.json
```

---

## 3. Stack B — Nuxt 3 (Full-Featured Vue Framework)

Nuxt 3 adds SSR, file-based routing, auto-imports, server functions, and SEO out of the box.

### Step 1 — Create the project

```bash
npm create nuxt@latest my-nuxt-app -- -t v3
```

### Step 2 — Install and run

```bash
cd my-nuxt-app
npm install
npm run dev -- -o
```

Opens automatically at `http://localhost:3000`.

### Step 3 — Build for production

```bash
npm run build      # SSR build (Node.js server)
# OR
npm run generate   # Static site generation (SSG)
```

### Key concepts

| Feature | Description |
|---------|-------------|
| **File-based routing** | Files in `pages/` become routes automatically |
| **Auto-imports** | Vue APIs, composables, and components are available without imports |
| **Server routes** | Files in `server/api/` become API endpoints (Nitro engine) |
| **Layouts** | Define shared page layouts in `layouts/` |
| **Middleware** | Route guards in `middleware/` |
| **SEO** | Built-in `useHead()` and `useSeoMeta()` composables |

### Project structure

```
my-nuxt-app/
├── pages/               # File-based routes
│   ├── index.vue        # → /
│   └── about.vue        # → /about
├── components/          # Auto-imported components
├── composables/         # Auto-imported composables
├── layouts/             # Page layouts
├── middleware/           # Route middleware
├── server/
│   └── api/             # Server API routes (Nitro)
├── public/              # Static assets
├── app.vue              # Root component
├── nuxt.config.ts       # Nuxt configuration
└── package.json
```

---

## 4. Stack C — Vue + Laravel (Inertia.js)

Inertia.js acts as a glue layer — no separate API needed. You get SPA feel with Laravel's full backend power.

### Step 1 — Create the Laravel project

```bash
composer create-project laravel/laravel my-laravel-vue
cd my-laravel-vue
```

### Step 2 — Install Inertia server-side

```bash
composer require inertiajs/inertia-laravel
php artisan inertia:middleware
```

Register the middleware in `bootstrap/app.php`.

### Step 3 — Install Inertia client-side + Vue

```bash
npm install @inertiajs/vue3 vue@latest
npm install -D @vitejs/plugin-vue
```

### Step 4 — Configure Vite

In `vite.config.js`:

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import laravel from 'laravel-vite-plugin'

export default defineConfig({
    plugins: [
        laravel({ input: ['resources/js/app.js'], refresh: true }),
        vue(),
    ],
})
```

### Step 5 — Set up the Vue entry point

In `resources/js/app.js`:

```javascript
import { createApp, h } from 'vue'
import { createInertiaApp } from '@inertiajs/vue3'

createInertiaApp({
    resolve: name => {
        const pages = import.meta.glob('./Pages/**/*.vue', { eager: true })
        return pages[`./Pages/${name}.vue`]
    },
    setup({ el, App, props, plugin }) {
        createApp({ render: () => h(App, props) })
            .use(plugin)
            .mount(el)
    },
})
```

### Step 6 — Create a page component

In `resources/js/Pages/Home.vue`:

```vue
<script setup>
defineProps({ greeting: String })
</script>

<template>
  <h1>{{ greeting }}</h1>
</template>
```

### Step 7 — Return from a Laravel route

In `routes/web.php`:

```php
use Inertia\Inertia;

Route::get('/', function () {
    return Inertia::render('Home', ['greeting' => 'Hello from Laravel!']);
});
```

### Step 8 — Run

```bash
php artisan serve   # Backend on :8000
npm run dev         # Vite on :5173
```

---

## 5. Stack D — Vue + Django REST Framework

A decoupled architecture: Django serves a REST API, Vue consumes it.

### Step 1 — Create the Django backend

```bash
pip install django djangorestframework django-cors-headers
django-admin startproject backend
cd backend
python manage.py startapp api
```

### Step 2 — Configure Django settings

In `backend/settings.py`, add to `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'corsheaders',
    'api',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    # ... other middleware
]

CORS_ALLOWED_ORIGINS = [
    'http://localhost:5173',
]
```

### Step 3 — Create a simple API

In `api/views.py`:

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET'])
def hello(request):
    return Response({'message': 'Hello from Django!'})
```

In `backend/urls.py`:

```python
from api.views import hello

urlpatterns = [
    path('api/hello/', hello),
]
```

### Step 4 — Create the Vue frontend (separate folder)

```bash
cd ..
npm create vue@latest frontend
cd frontend
npm install
```

### Step 5 — Fetch from the API

In a Vue component:

```vue
<script setup>
import { ref, onMounted } from 'vue'

const message = ref('')

onMounted(async () => {
    const res = await fetch('http://localhost:8000/api/hello/')
    const data = await res.json()
    message.value = data.message
})
</script>

<template>
  <p>{{ message }}</p>
</template>
```

### Step 6 — Run both

```bash
# Terminal 1 — Django
cd backend && python manage.py runserver

# Terminal 2 — Vue
cd frontend && npm run dev
```

---

## 6. Stack E — Vue + Phoenix / Elixir (LiveVue)

LiveVue bridges Phoenix LiveView and Vue.js — server-driven reactivity with rich client-side components.

### Requirements

- Elixir 1.16+
- Phoenix 1.8+
- Node.js 20+

### Step 1 — Create a Phoenix project

```bash
mix phx.new my_app --live
cd my_app
mix deps.get
```

### Step 2 — Install LiveVue

```bash
mix igniter.install live_vue
```

This adds the Hex dependency, configures Vite, and sets up the Vue integration automatically.

### Step 3 — Create a Vue component

In `assets/vue/components/Counter.vue`:

```vue
<script setup>
import { useLiveVue } from 'live_vue'

const props = defineProps({ count: Number })
const { pushEvent } = useLiveVue()
</script>

<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="pushEvent('increment')">+1</button>
  </div>
</template>
```

### Step 4 — Use it in a LiveView

In `lib/my_app_web/live/counter_live.ex`:

```elixir
defmodule MyAppWeb.CounterLive do
  use MyAppWeb, :live_view

  def mount(_params, _session, socket) do
    {:ok, assign(socket, count: 0)}
  end

  def handle_event("increment", _params, socket) do
    {:noreply, update(socket, :count, &(&1 + 1))}
  end

  def render(assigns) do
    ~H"""
    <.vue component="Counter" count={@count} v-on:increment="increment" />
    """
  end
end
```

### Step 5 — Run

```bash
mix phx.server
```

### Key features

- LiveView assigns become Vue props automatically
- Vue events trigger LiveView `handle_event/3`
- Server-side rendering on first paint (SEO-friendly)
- Composables: `useLiveVue()`, `useLiveForm()`, `useLiveUpload()`, `useLiveEvent()`
- HMR via Vite across the full stack

---

## 7. Stack F — Vue + Electron (Desktop)

Build cross-platform desktop apps with Vue.js and Electron (Chromium + Node.js).

### Step 1 — Create a Vue project

```bash
npm create vue@latest my-desktop-app
cd my-desktop-app
npm install
```

### Step 2 — Add Electron

```bash
npm install -D electron electron-builder concurrently wait-on
```

### Step 3 — Create the Electron main process

Create `electron/main.js`:

```javascript
const { app, BrowserWindow } = require('electron')
const path = require('path')

function createWindow() {
    const win = new BrowserWindow({
        width: 1200,
        height: 800,
        webPreferences: {
            preload: path.join(__dirname, 'preload.js'),
        },
    })

    if (process.env.VITE_DEV_SERVER_URL) {
        win.loadURL(process.env.VITE_DEV_SERVER_URL)
    } else {
        win.loadFile(path.join(__dirname, '../dist/index.html'))
    }
}

app.whenReady().then(createWindow)
app.on('window-all-closed', () => {
    if (process.platform !== 'darwin') app.quit()
})
```

### Step 4 — Update package.json

```json
{
  "main": "electron/main.js",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "electron:dev": "concurrently \"vite\" \"wait-on http://localhost:5173 && electron .\"",
    "electron:build": "vite build && electron-builder"
  }
}
```

### Step 5 — Configure Vite for Electron

In `vite.config.ts`, set `base: './'` so paths resolve correctly in the packaged app.

### Step 6 — Run

```bash
npm run electron:dev
```

### Packaging

```bash
npm run electron:build
```

Produces installers for Windows (.exe), macOS (.dmg), and Linux (.AppImage).

---

## 8. Stack G — Vue + Tauri 2 (Lightweight Desktop)

Tauri uses the OS native WebView instead of bundling Chromium — resulting in apps as small as ~600 KB.

### Requirements

- Rust toolchain (`rustup` — install from [rustup.rs](https://rustup.rs/))
- Node.js 20+
- OS-specific dependencies (see [Tauri prerequisites](https://tauri.app/start/prerequisites/))

### Step 1 — Scaffold the project

```bash
sh <(curl https://create.tauri.app/sh)
```

Select **Vue** as the frontend framework when prompted. On Windows use:

```powershell
irm https://create.tauri.app/ps | iex
```

### Step 2 — Install and run

```bash
cd tauri-app
npm install
npm run tauri dev
```

A native window opens with your Vue app running inside.

### Step 3 — Communicate between Vue and Rust

In a Vue component:

```vue
<script setup>
import { invoke } from '@tauri-apps/api/core'
import { ref } from 'vue'

const greeting = ref('')

async function greet() {
    greeting.value = await invoke('greet', { name: 'World' })
}
</script>

<template>
  <button @click="greet">Greet</button>
  <p>{{ greeting }}</p>
</template>
```

The Rust backend (`src-tauri/src/lib.rs`):

```rust
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}! From Rust.", name)
}
```

### Step 4 — Build for production

```bash
npm run tauri build
```

### Tauri vs Electron

| | Tauri 2 | Electron |
|--|---------|----------|
| Bundle size | ~600 KB – 5 MB | ~100 MB+ |
| Memory usage | ~50 MB | ~150 MB+ |
| Backend language | Rust | Node.js |
| Rendering engine | OS native WebView | Bundled Chromium |
| Startup time | < 1 second | 2–5 seconds |

---

## 9. Core Concepts Cheat Sheet

### Composition API (the modern way)

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
    count.value++
}

onMounted(() => {
    console.log('Component mounted')
})
</script>

<template>
  <button @click="increment">{{ count }} (× 2 = {{ doubled }})</button>
</template>
```

### Key APIs at a glance

| API | Purpose |
|-----|---------|
| `ref()` | Reactive primitive value |
| `reactive()` | Reactive object |
| `computed()` | Derived/cached value |
| `watch()` / `watchEffect()` | React to state changes |
| `onMounted()` | Lifecycle — after DOM insertion |
| `onUnmounted()` | Lifecycle — before removal |
| `defineProps()` | Declare component props |
| `defineEmits()` | Declare component events |
| `provide()` / `inject()` | Dependency injection across component tree |

### State management (Pinia)

```vue
<!-- stores/counter.ts -->
<script>
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useCounterStore = defineStore('counter', () => {
    const count = ref(0)
    function increment() { count.value++ }
    return { count, increment }
})
</script>
```

### Routing (Vue Router)

```typescript
import { createRouter, createWebHistory } from 'vue-router'
import Home from '@/views/Home.vue'
import About from '@/views/About.vue'

const router = createRouter({
    history: createWebHistory(),
    routes: [
        { path: '/', component: Home },
        { path: '/about', component: About },
    ],
})

export default router
```

---

## 10. Recommended Extensions & Tooling

### VS Code / Cursor

| Extension | Purpose |
|-----------|---------|
| **Vue - Official** (`Vue.volar`) | Syntax highlighting, type checking, IntelliSense |
| **TypeScript Vue Plugin** | Enhanced TS support in `.vue` files |
| **ESLint** | Linting |
| **Prettier** | Code formatting |

### Browser

| Tool | Purpose |
|------|---------|
| **Vue DevTools** | Component inspector, Pinia state viewer, router debugging |

### CLI Commands Reference

| Command | What it does |
|---------|-------------|
| `npm create vue@latest` | Scaffold a new Vue + Vite project |
| `npm create nuxt@latest` | Scaffold a new Nuxt 3 project |
| `npm run dev` | Start Vite dev server with HMR |
| `npm run build` | Production build |
| `npm run preview` | Preview production build locally |
| `npm run test:unit` | Run Vitest unit tests |
| `npm run lint` | Run ESLint |

---

**Last updated:** March 2026
