# TypeScript — Zero to Hero

> A typed superset of JavaScript that compiles to clean, readable JS.
> Current version: **5.9** | License: Apache-2.0

---

## Table of Contents

1. [What Is TypeScript](#1-what-is-typescript)
2. [Installation](#2-installation)
3. [Basic Types](#3-basic-types)
4. [Functions](#4-functions)
5. [Interfaces & Type Aliases](#5-interfaces--type-aliases)
6. [Unions, Intersections & Narrowing](#6-unions-intersections--narrowing)
7. [Generics](#7-generics)
8. [Enums](#8-enums)
9. [Utility Types](#9-utility-types)
10. [Classes](#10-classes)
11. [Modules & Namespaces](#11-modules--namespaces)
12. [Decorators](#12-decorators)
13. [Type Guards & Assertions](#13-type-guards--assertions)
14. [Advanced Patterns](#14-advanced-patterns)
15. [tsconfig.json Deep Dive](#15-tsconfigjson-deep-dive)
16. [TypeScript with Frameworks](#16-typescript-with-frameworks)

---

## 1. What Is TypeScript

TypeScript adds **optional static types** to JavaScript. Types are checked at compile time and completely erased in the output — zero runtime cost. Benefits:

- Catch bugs before they reach production
- Rich IDE autocompletion and refactoring
- Self-documenting code via type annotations
- Gradual adoption — any valid JS is valid TS

---

## 2. Installation

### Global

```bash
npm install -g typescript
tsc --version
```

### Per-project (recommended)

```bash
npm install -D typescript
npx tsc --init           # generates tsconfig.json
```

### Compile and run

```bash
npx tsc                  # compile all files per tsconfig
npx tsc src/index.ts     # compile a single file
node dist/index.js       # run the output
```

### Run directly (no compile step)

```bash
npm install -D tsx
npx tsx src/index.ts
```

Or use `ts-node`:

```bash
npm install -D ts-node
npx ts-node src/index.ts
```

---

## 3. Basic Types

### Primitives

```typescript
const name: string = 'Alice'
const age: number = 30
const active: boolean = true
const nothing: null = null
const notDefined: undefined = undefined
const uniqueId: symbol = Symbol('id')
const huge: bigint = 9007199254740991n
```

### Arrays

```typescript
const numbers: number[] = [1, 2, 3]
const names: Array<string> = ['Alice', 'Bob']
```

### Tuples

```typescript
const pair: [string, number] = ['age', 30]
const namedTuple: [name: string, age: number] = ['Alice', 30]
```

### Special types

```typescript
let anything: any = 'no type checking'
let unknown: unknown = 'safe any — must narrow before use'
function fail(): never { throw new Error('crash') }
function log(): void { console.log('no return value') }
```

### Literal types

```typescript
type Direction = 'north' | 'south' | 'east' | 'west'
type Digit = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

---

## 4. Functions

### Typed parameters and return

```typescript
function add(a: number, b: number): number {
  return a + b
}
```

### Optional and default parameters

```typescript
function greet(name: string, greeting: string = 'Hello'): string {
  return `${greeting}, ${name}`
}

function greetOptional(name: string, greeting?: string): string {
  return `${greeting ?? 'Hi'}, ${name}`
}
```

### Rest parameters

```typescript
function sum(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0)
}
```

### Function types

```typescript
type Comparator<T> = (a: T, b: T) => number
const numSort: Comparator<number> = (a, b) => a - b
```

### Overloads

```typescript
function parse(input: string): object
function parse(input: object): string
function parse(input: string | object): string | object {
  return typeof input === 'string' ? JSON.parse(input) : JSON.stringify(input)
}
```

---

## 5. Interfaces & Type Aliases

### Interfaces

```typescript
interface User {
  id: number
  name: string
  email: string
  avatar?: string            // optional
  readonly createdAt: Date   // immutable
}
```

### Extending interfaces

```typescript
interface Admin extends User {
  role: 'admin'
  permissions: string[]
}
```

### Type aliases

```typescript
type Point = { x: number; y: number }
type ID = string | number
type Handler = (event: Event) => void
```

### When to use which

| Feature | `interface` | `type` |
|---------|-------------|--------|
| Extend / implement | `extends` / `implements` | Intersection `&` |
| Declaration merging | Yes | No |
| Unions | No | Yes |
| Mapped types | No | Yes |
| Primitives / tuples | No | Yes |

**Rule of thumb:** Use `interface` for object shapes, `type` for everything else.

---

## 6. Unions, Intersections & Narrowing

### Union types (OR)

```typescript
type Result = string | number

function format(value: Result): string {
  if (typeof value === 'string') return value.toUpperCase()
  return value.toFixed(2)
}
```

### Intersection types (AND)

```typescript
type Timestamped = { createdAt: Date; updatedAt: Date }
type TimestampedUser = User & Timestamped
```

### Discriminated unions

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rect'; width: number; height: number }

function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.radius ** 2
    case 'rect': return s.width * s.height
  }
}
```

---

## 7. Generics

### Generic functions

```typescript
function identity<T>(value: T): T {
  return value
}

const str = identity('hello')   // type: string
const num = identity(42)        // type: number
```

### Generic constraints

```typescript
function getLength<T extends { length: number }>(item: T): number {
  return item.length
}
```

### Generic interfaces

```typescript
interface Repository<T> {
  findById(id: string): Promise<T>
  save(entity: T): Promise<void>
  delete(id: string): Promise<boolean>
}
```

### Multiple type parameters

```typescript
function merge<A, B>(a: A, b: B): A & B {
  return { ...a, ...b }
}
```

### Generic defaults

```typescript
interface ApiResponse<T = unknown> {
  data: T
  status: number
}
```

---

## 8. Enums

### Numeric enums

```typescript
enum Status {
  Pending,      // 0
  Active,       // 1
  Archived,     // 2
}
```

### String enums

```typescript
enum Color {
  Red = '#FF0000',
  Green = '#00FF00',
  Blue = '#0000FF',
}
```

### Const enums (inlined at compile time)

```typescript
const enum HttpMethod {
  GET = 'GET',
  POST = 'POST',
  PUT = 'PUT',
  DELETE = 'DELETE',
}
```

### Alternative: union of literals (often preferred)

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE'
```

---

## 9. Utility Types

### Transforming properties

| Utility | Effect |
|---------|--------|
| `Partial<T>` | All properties optional |
| `Required<T>` | All properties required |
| `Readonly<T>` | All properties readonly |
| `Mutable<T>` | Remove readonly (TS 5.9+) |

```typescript
interface User { name: string; email: string }
type PartialUser = Partial<User>     // { name?: string; email?: string }
```

### Picking & omitting

```typescript
type UserPreview = Pick<User, 'name'>
type UserWithoutEmail = Omit<User, 'email'>
```

### Record

```typescript
type RolePermissions = Record<'admin' | 'editor' | 'viewer', string[]>
```

### Extract & Exclude

```typescript
type T = 'a' | 'b' | 'c'
type OnlyAB = Extract<T, 'a' | 'b'>     // 'a' | 'b'
type NoA = Exclude<T, 'a'>               // 'b' | 'c'
```

### Function utilities

```typescript
type Params = Parameters<typeof fetch>          // [input, init?]
type Return = ReturnType<typeof fetch>           // Promise<Response>
type Awaited = Awaited<Promise<string>>          // string
```

### NonNullable

```typescript
type MaybeString = string | null | undefined
type SafeString = NonNullable<MaybeString>       // string
```

---

## 10. Classes

```typescript
class Animal {
  readonly species: string
  private _name: string
  protected age: number

  constructor(species: string, name: string, age: number) {
    this.species = species
    this._name = name
    this.age = age
  }

  get name(): string {
    return this._name
  }

  speak(): string {
    return `${this._name} makes a sound`
  }
}

class Dog extends Animal {
  constructor(name: string, age: number) {
    super('Canis lupus familiaris', name, age)
  }

  speak(): string {
    return `${this.name} barks`
  }
}
```

### Abstract classes

```typescript
abstract class Shape {
  abstract area(): number

  describe(): string {
    return `Shape with area ${this.area()}`
  }
}
```

### Implementing interfaces

```typescript
interface Serializable {
  toJSON(): string
}

class Config implements Serializable {
  constructor(private data: Record<string, unknown>) {}

  toJSON(): string {
    return JSON.stringify(this.data)
  }
}
```

---

## 11. Modules & Namespaces

### ES Modules (the standard)

```typescript
// math.ts
export function add(a: number, b: number) { return a + b }
export default class Calculator { /* ... */ }

// main.ts
import Calculator, { add } from './math'
```

### Re-exports (barrel files)

```typescript
// index.ts
export { UserService } from './user.service'
export { AuthService } from './auth.service'
export type { User } from './user.types'
```

### Type-only imports

```typescript
import type { User } from './user.types'
```

Erased at compile time — never included in the JS output.

---

## 12. Decorators

Stabilized in TS 5.9 via TC39 Stage 3 Decorator Metadata.

```typescript
function Log(target: any, context: ClassMethodDecoratorContext) {
  const original = target
  return function (this: any, ...args: any[]) {
    console.log(`Calling ${String(context.name)} with`, args)
    return original.apply(this, args)
  }
}

class Service {
  @Log
  fetchData(id: string) {
    return { id }
  }
}
```

Enable in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "experimentalDecorators": false,
    "emitDecoratorMetadata": false
  }
}
```

---

## 13. Type Guards & Assertions

### typeof guard

```typescript
function double(value: string | number) {
  if (typeof value === 'string') return value.repeat(2)
  return value * 2
}
```

### instanceof guard

```typescript
function handleError(err: Error | string) {
  if (err instanceof Error) return err.message
  return err
}
```

### Custom type guard

```typescript
interface Fish { swim(): void }
interface Bird { fly(): void }

function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined
}
```

### Type assertion

```typescript
const input = document.getElementById('name') as HTMLInputElement
input.value = 'Alice'
```

### Satisfies (TS 4.9+)

```typescript
const palette = {
  red: [255, 0, 0],
  green: '#00FF00',
} satisfies Record<string, string | number[]>
```

Validates the type without widening — you keep the precise inferred type.

---

## 14. Advanced Patterns

### Mapped types

```typescript
type Optional<T> = { [K in keyof T]?: T[K] }
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] }
```

### Conditional types

```typescript
type IsString<T> = T extends string ? true : false
type A = IsString<'hello'>   // true
type B = IsString<42>        // false
```

### Template literal types

```typescript
type EventName = `on${Capitalize<'click' | 'focus' | 'blur'>}`
// 'onClick' | 'onFocus' | 'onBlur'
```

### Infer keyword

```typescript
type UnpackPromise<T> = T extends Promise<infer U> ? U : T
type Result = UnpackPromise<Promise<string>>   // string
```

### Recursive types

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K]
}
```

---

## 15. tsconfig.json Deep Dive

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,

    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,

    "esModuleInterop": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "verbatimModuleSyntax": true,

    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Key flags explained

| Flag | Purpose |
|------|---------|
| `strict` | Enables all strict checks (nulls, any, bind, etc.) |
| `noUncheckedIndexedAccess` | Array/object index access returns `T \| undefined` |
| `verbatimModuleSyntax` | Enforces `import type` for type-only imports |
| `moduleResolution: "bundler"` | Matches how Vite/webpack resolve modules |
| `isolatedModules` | Required for single-file transpilers (Vite, esbuild) |

---

## 16. TypeScript with Frameworks

### With Vite

Vite transpiles TypeScript out of the box via esbuild — no plugin needed. Add type checking as a separate step:

```bash
npx tsc --noEmit
```

### With Vue

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref<number>(0)

interface Props {
  title: string
  count?: number
}
const props = defineProps<Props>()
</script>
```

### With React

```tsx
interface ButtonProps {
  label: string
  onClick: () => void
  variant?: 'primary' | 'secondary'
}

function Button({ label, onClick, variant = 'primary' }: ButtonProps) {
  return <button className={variant} onClick={onClick}>{label}</button>
}
```

### With Express

```typescript
import express, { Request, Response, NextFunction } from 'express'

const app = express()

app.get('/api/users/:id', (req: Request, res: Response) => {
  const { id } = req.params
  res.json({ id })
})
```

---

**Last updated:** March 2026
