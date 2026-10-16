# Java Internal Features — Built-in Functions & Functional Language Features (11+)

A comprehensive cheatsheet covering Java's built-in utility methods and functional programming capabilities from **Java 11** through **Java 21+**.

---

## Table of Contents

- [String Methods](#string-methods)
- [Collection Factory Methods](#collection-factory-methods)
- [Optional](#optional)
- [Functional Interfaces](#functional-interfaces)
- [Lambda Expressions](#lambda-expressions)
- [Method References](#method-references)
- [Stream API](#stream-api)
- [Collectors](#collectors)
- [Map Operations](#map-operations)
- [CompletableFuture](#completablefuture)
- [Records](#records)
- [Sealed Classes](#sealed-classes)
- [Pattern Matching](#pattern-matching)
- [Switch Expressions](#switch-expressions)
- [Text Blocks](#text-blocks)
- [var (Local Variable Type Inference)](#var-local-variable-type-inference)
- [New I/O & HTTP Methods](#new-io--http-methods)
- [Helpful NullPointerExceptions](#helpful-nullpointerexceptions)
- [Version Reference](#version-reference)

---

## String Methods

### Since Java 11

| Method | Returns | Description |
| :--- | :--- | :--- |
| `isBlank()` | `boolean` | `true` if empty or contains only whitespace |
| `strip()` | `String` | Removes leading and trailing whitespace (Unicode-aware) |
| `stripLeading()` | `String` | Removes leading whitespace |
| `stripTrailing()` | `String` | Removes trailing whitespace |
| `lines()` | `Stream<String>` | Splits by line terminators into a stream |
| `repeat(int)` | `String` | Repeats the string n times |

```java
"  hello  ".strip();          // "hello"
"  ".isBlank();               // true
"ab".repeat(3);               // "ababab"
"line1\nline2".lines().count(); // 2
```

### Since Java 12

| Method | Returns | Description |
| :--- | :--- | :--- |
| `indent(int)` | `String` | Adjusts indentation by n spaces |
| `transform(Function)` | `R` | Applies a function to the string and returns the result |

```java
"hello".indent(4);                     // "    hello\n"
"hello".transform(String::toUpperCase); // "HELLO"
```

### Since Java 15

| Method | Returns | Description |
| :--- | :--- | :--- |
| `formatted(Object...)` | `String` | Equivalent to `String.format(this, args)` |
| `stripIndent()` | `String` | Removes incidental whitespace from multi-line strings |
| `translateEscapes()` | `String` | Translates escape sequences like `\\n` to `\n` |

```java
"Hello %s, you are %d".formatted("Alice", 30); // "Hello Alice, you are 30"
```

---

## Collection Factory Methods

Immutable collection creation (since Java 9, widely used from 11+).

| Factory | Returns | Example |
| :--- | :--- | :--- |
| `List.of(...)` | `List<T>` | `List.of("a", "b", "c")` |
| `Set.of(...)` | `Set<T>` | `Set.of(1, 2, 3)` |
| `Map.of(k, v, ...)` | `Map<K,V>` | `Map.of("a", 1, "b", 2)` |
| `Map.ofEntries(...)` | `Map<K,V>` | `Map.ofEntries(entry("a", 1))` |
| `List.copyOf(coll)` | `List<T>` | Immutable copy of an existing collection |
| `Set.copyOf(coll)` | `Set<T>` | Immutable copy, duplicates removed |
| `Map.copyOf(map)` | `Map<K,V>` | Immutable copy of an existing map |

```java
var names = List.of("Alice", "Bob", "Charlie");
var lookup = Map.of("host", "localhost", "port", "8080");

// copyOf since Java 10
var copy = List.copyOf(mutableList);
```

> [!WARNING]
> These collections are **unmodifiable**. Calling `add()`, `put()`, or `remove()` throws `UnsupportedOperationException`. They also reject `null` elements.

---

## Optional

Container for a value that may or may not be present.

### Creation

| Method | Description |
| :--- | :--- |
| `Optional.of(value)` | Wraps a non-null value |
| `Optional.ofNullable(value)` | Wraps a value that may be null |
| `Optional.empty()` | Returns an empty Optional |

### Core Operations

| Method | Returns | Description |
| :--- | :--- | :--- |
| `isPresent()` | `boolean` | `true` if value is present |
| `isEmpty()` | `boolean` | `true` if no value (since Java 11) |
| `get()` | `T` | Returns value or throws `NoSuchElementException` |
| `orElse(default)` | `T` | Returns value or the default |
| `orElseGet(Supplier)` | `T` | Returns value or calls the supplier |
| `orElseThrow()` | `T` | Returns value or throws `NoSuchElementException` (since Java 10) |
| `orElseThrow(Supplier)` | `T` | Returns value or throws supplied exception |
| `or(Supplier<Optional>)` | `Optional<T>` | Returns this if present, otherwise the supplied Optional |
| `ifPresent(Consumer)` | `void` | Executes if value is present |
| `ifPresentOrElse(Consumer, Runnable)` | `void` | Executes consumer if present, runnable if empty |
| `map(Function)` | `Optional<U>` | Transforms value if present |
| `flatMap(Function)` | `Optional<U>` | Transforms and flattens nested Optional |
| `filter(Predicate)` | `Optional<T>` | Returns this if value matches predicate |
| `stream()` | `Stream<T>` | Zero or one element stream |

```java
Optional.ofNullable(name)
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .orElse("UNKNOWN");

Optional.ofNullable(user)
    .ifPresentOrElse(
        u -> System.out.println(u.name()),
        () -> System.out.println("No user found")
    );

// Chaining fallback Optionals
Optional<String> result = findInCache(key)
    .or(() -> findInDatabase(key))
    .or(() -> Optional.of("default"));
```

---

## Functional Interfaces

Built-in interfaces in `java.util.function`. All are `@FunctionalInterface`.

### Core Four

| Interface | Method | Signature | Purpose |
| :--- | :--- | :--- | :--- |
| `Function<T,R>` | `apply(T)` | `T → R` | Transform a value |
| `Predicate<T>` | `test(T)` | `T → boolean` | Test a condition |
| `Consumer<T>` | `accept(T)` | `T → void` | Perform a side effect |
| `Supplier<T>` | `get()` | `() → T` | Produce a value |

### Extended Variants

| Interface | Method | Signature |
| :--- | :--- | :--- |
| `BiFunction<T,U,R>` | `apply(T, U)` | `(T, U) → R` |
| `BiPredicate<T,U>` | `test(T, U)` | `(T, U) → boolean` |
| `BiConsumer<T,U>` | `accept(T, U)` | `(T, U) → void` |
| `UnaryOperator<T>` | `apply(T)` | `T → T` (specialisation of Function) |
| `BinaryOperator<T>` | `apply(T, T)` | `(T, T) → T` (specialisation of BiFunction) |

### Primitive Specialisations (avoid boxing)

| Interface | Method | Signature |
| :--- | :--- | :--- |
| `IntFunction<R>` | `apply(int)` | `int → R` |
| `IntPredicate` | `test(int)` | `int → boolean` |
| `IntConsumer` | `accept(int)` | `int → void` |
| `IntSupplier` | `getAsInt()` | `() → int` |
| `IntUnaryOperator` | `applyAsInt(int)` | `int → int` |
| `IntBinaryOperator` | `applyAsInt(int, int)` | `(int, int) → int` |
| `ToIntFunction<T>` | `applyAsInt(T)` | `T → int` |

> [!NOTE]
> Equivalent `Long` and `Double` variants exist for all primitive specialisations (`LongFunction`, `DoubleConsumer`, etc.).

### Composition Methods

| Method | Available On | Description |
| :--- | :--- | :--- |
| `andThen(after)` | `Function`, `Consumer` | Chain: apply this, then apply after |
| `compose(before)` | `Function` | Chain: apply before first, then this |
| `and(other)` | `Predicate` | Logical AND |
| `or(other)` | `Predicate` | Logical OR |
| `negate()` | `Predicate` | Logical NOT |
| `not(Predicate)` | `Predicate` (static, since Java 11) | Static negation factory |

```java
Predicate<String> notBlank = Predicate.not(String::isBlank);

Function<String, String> trimAndUpper =
    String::strip.andThen(String::toUpperCase);

Predicate<Integer> between = ((Predicate<Integer>) i -> i > 0).and(i -> i < 100);
```

---

## Lambda Expressions

Concise syntax for implementing functional interfaces.

### Syntax Forms

```java
// Full form
(String a, String b) -> { return a.compareTo(b); }

// Type inference
(a, b) -> { return a.compareTo(b); }

// Single expression (implicit return)
(a, b) -> a.compareTo(b)

// Single parameter (no parentheses)
x -> x * 2

// No parameters
() -> System.out.println("Hello")

// var in parameters (since Java 11) — allows annotations
(@NonNull var x, @NonNull var y) -> x + y
```

### Effectively Final Requirement

Lambdas can capture local variables only if they are **effectively final** (never reassigned after initialisation).

```java
String prefix = "Hello";   // effectively final
Function<String, String> greet = name -> prefix + " " + name;
```

---

## Method References

Shorthand for lambdas that call a single existing method.

| Type | Syntax | Lambda Equivalent |
| :--- | :--- | :--- |
| Static method | `ClassName::staticMethod` | `x -> ClassName.staticMethod(x)` |
| Instance method (bound) | `instance::method` | `x -> instance.method(x)` |
| Instance method (unbound) | `ClassName::method` | `(obj, x) -> obj.method(x)` |
| Constructor | `ClassName::new` | `x -> new ClassName(x)` |

```java
list.stream().map(String::toUpperCase);         // unbound instance
list.stream().forEach(System.out::println);     // bound instance
list.stream().map(Integer::parseInt);           // static
list.stream().map(User::new);                   // constructor
```

---

## Stream API

### Creating Streams

| Method | Source | Example |
| :--- | :--- | :--- |
| `collection.stream()` | Any Collection | `list.stream()` |
| `collection.parallelStream()` | Any Collection | `list.parallelStream()` |
| `Arrays.stream(array)` | Array | `Arrays.stream(arr)` |
| `Stream.of(...)` | Varargs | `Stream.of("a", "b")` |
| `Stream.empty()` | Nothing | `Stream.empty()` |
| `Stream.iterate(seed, f)` | Infinite | `Stream.iterate(0, i -> i + 1)` |
| `Stream.iterate(seed, pred, f)` | Bounded (since Java 9) | `Stream.iterate(0, i -> i < 10, i -> i + 1)` |
| `Stream.generate(Supplier)` | Infinite | `Stream.generate(Math::random)` |
| `Stream.concat(a, b)` | Two streams | `Stream.concat(s1, s2)` |
| `string.chars()` | String | `"hello".chars()` (IntStream) |
| `IntStream.range(start, end)` | Range (exclusive) | `IntStream.range(0, 10)` |
| `IntStream.rangeClosed(start, end)` | Range (inclusive) | `IntStream.rangeClosed(1, 10)` |

### Intermediate Operations (lazy, return Stream)

| Operation | Signature | Description |
| :--- | :--- | :--- |
| `filter(Predicate)` | `T → boolean` | Keep elements matching the predicate |
| `map(Function)` | `T → R` | Transform each element |
| `flatMap(Function)` | `T → Stream<R>` | Transform and flatten nested streams |
| `mapMulti(BiConsumer)` | Since Java 16 | Imperative alternative to flatMap |
| `distinct()` | | Remove duplicates (by `equals`) |
| `sorted()` | | Natural order |
| `sorted(Comparator)` | | Custom order |
| `peek(Consumer)` | | Side effect without consuming (debugging) |
| `limit(long)` | | Take first n elements |
| `skip(long)` | | Skip first n elements |
| `takeWhile(Predicate)` | Since Java 9 | Take while predicate is true |
| `dropWhile(Predicate)` | Since Java 9 | Skip while predicate is true |

### Terminal Operations (trigger execution)

| Operation | Returns | Description |
| :--- | :--- | :--- |
| `forEach(Consumer)` | `void` | Execute action for each element |
| `forEachOrdered(Consumer)` | `void` | Ordered execution (parallel safe) |
| `toArray()` | `Object[]` | Collect into an array |
| `toArray(IntFunction)` | `T[]` | Collect into a typed array |
| `toList()` | `List<T>` | Unmodifiable list (since Java 16) |
| `reduce(identity, BinaryOp)` | `T` | Fold into a single value |
| `reduce(BinaryOp)` | `Optional<T>` | Fold without identity |
| `collect(Collector)` | `R` | Mutable reduction (see Collectors) |
| `count()` | `long` | Number of elements |
| `min(Comparator)` | `Optional<T>` | Minimum by comparator |
| `max(Comparator)` | `Optional<T>` | Maximum by comparator |
| `anyMatch(Predicate)` | `boolean` | At least one matches |
| `allMatch(Predicate)` | `boolean` | All match |
| `noneMatch(Predicate)` | `boolean` | None match |
| `findFirst()` | `Optional<T>` | First element |
| `findAny()` | `Optional<T>` | Any element (non-deterministic in parallel) |

### Stream Pipeline Examples

```java
// Filter, transform, collect
List<String> upperNames = people.stream()
    .filter(p -> p.age() >= 18)
    .map(Person::name)
    .map(String::toUpperCase)
    .sorted()
    .toList();                          // since Java 16

// Reduce
int total = numbers.stream()
    .reduce(0, Integer::sum);

// FlatMap — flatten nested lists
List<String> allTags = posts.stream()
    .flatMap(post -> post.tags().stream())
    .distinct()
    .toList();

// takeWhile / dropWhile
List<Integer> prefix = Stream.of(1, 2, 3, 5, 4)
    .takeWhile(n -> n < 5)
    .toList();                          // [1, 2, 3]

// Parallel stream
long count = bigList.parallelStream()
    .filter(item -> item.isValid())
    .count();
```

---

## Collectors

Predefined collectors in `java.util.stream.Collectors`.

### Collection Collectors

| Collector | Returns | Description |
| :--- | :--- | :--- |
| `toList()` | `List<T>` | Mutable ArrayList |
| `toUnmodifiableList()` | `List<T>` | Immutable list |
| `toSet()` | `Set<T>` | Mutable HashSet |
| `toUnmodifiableSet()` | `Set<T>` | Immutable set |
| `toCollection(Supplier)` | `C` | Custom collection type |

### Map Collectors

| Collector | Returns | Description |
| :--- | :--- | :--- |
| `toMap(keyFn, valFn)` | `Map<K,V>` | Key-value map (throws on duplicate keys) |
| `toMap(keyFn, valFn, mergeFn)` | `Map<K,V>` | With merge strategy for duplicates |
| `toUnmodifiableMap(...)` | `Map<K,V>` | Immutable map |
| `toConcurrentMap(...)` | `ConcurrentMap<K,V>` | Thread-safe map |

### Grouping & Partitioning

| Collector | Returns | Description |
| :--- | :--- | :--- |
| `groupingBy(classifier)` | `Map<K, List<T>>` | Group by key |
| `groupingBy(classifier, downstream)` | `Map<K, R>` | Group and aggregate |
| `partitioningBy(predicate)` | `Map<Boolean, List<T>>` | Split into two groups |

### Aggregation Collectors

| Collector | Returns | Description |
| :--- | :--- | :--- |
| `counting()` | `Long` | Count elements |
| `summingInt(fn)` | `Integer` | Sum an int property |
| `averagingDouble(fn)` | `Double` | Average a numeric property |
| `summarizingInt(fn)` | `IntSummaryStatistics` | Count, sum, min, max, average |
| `maxBy(comparator)` | `Optional<T>` | Maximum |
| `minBy(comparator)` | `Optional<T>` | Minimum |
| `reducing(...)` | `T` | General reduction |

### String Collectors

| Collector | Returns | Description |
| :--- | :--- | :--- |
| `joining()` | `String` | Concatenate |
| `joining(delimiter)` | `String` | Concatenate with delimiter |
| `joining(delim, prefix, suffix)` | `String` | With prefix and suffix |

### Adaptor Collectors

| Collector | Returns | Description |
| :--- | :--- | :--- |
| `mapping(fn, downstream)` | `R` | Map before collecting |
| `filtering(pred, downstream)` | `R` | Filter before collecting (since Java 9) |
| `flatMapping(fn, downstream)` | `R` | FlatMap before collecting (since Java 9) |
| `collectingAndThen(downstream, fn)` | `R` | Post-process result |
| `teeing(c1, c2, mergeFn)` | `R` | Two collectors in parallel, merge results (since Java 12) |

### Collector Examples

```java
// Group people by city
Map<String, List<Person>> byCity = people.stream()
    .collect(Collectors.groupingBy(Person::city));

// Group and count
Map<String, Long> countByCity = people.stream()
    .collect(Collectors.groupingBy(Person::city, Collectors.counting()));

// Partition adults vs minors
Map<Boolean, List<Person>> partitioned = people.stream()
    .collect(Collectors.partitioningBy(p -> p.age() >= 18));

// Join names
String names = people.stream()
    .map(Person::name)
    .collect(Collectors.joining(", ", "[", "]"));  // [Alice, Bob, Charlie]

// Teeing — compute min and max in one pass (since Java 12)
record MinMax(int min, int max) {}
MinMax result = numbers.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Comparator.naturalOrder()),
        Collectors.maxBy(Comparator.naturalOrder()),
        (min, max) -> new MinMax(min.orElse(0), max.orElse(0))
    ));

// toMap with merge strategy
Map<String, Integer> scores = entries.stream()
    .collect(Collectors.toMap(
        Entry::key,
        Entry::value,
        Integer::sum
    ));
```

---

## Map Operations

Functional methods on `java.util.Map` (since Java 8, commonly used from 11+).

| Method | Description |
| :--- | :--- |
| `getOrDefault(key, default)` | Returns value or default if key absent |
| `putIfAbsent(key, value)` | Inserts only if key is not already present |
| `computeIfAbsent(key, Function)` | Compute and insert if key is absent |
| `computeIfPresent(key, BiFunction)` | Recompute if key is present |
| `compute(key, BiFunction)` | Compute regardless of presence |
| `merge(key, value, BiFunction)` | Merge value with existing |
| `replaceAll(BiFunction)` | Replace all values |
| `forEach(BiConsumer)` | Iterate over entries |

```java
// Frequency counter
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.merge(word, 1, Integer::sum);
}

// Lazy initialisation of list values
Map<String, List<String>> index = new HashMap<>();
index.computeIfAbsent("key", k -> new ArrayList<>()).add("value");

// Default value
String host = config.getOrDefault("host", "localhost");
```

---

## CompletableFuture

Composable asynchronous programming.

### Creation

| Method | Description |
| :--- | :--- |
| `supplyAsync(Supplier)` | Run task returning a value |
| `runAsync(Runnable)` | Run task with no return value |
| `completedFuture(value)` | Already completed with a value |
| `failedFuture(ex)` | Already completed with an exception (since Java 9) |

### Chaining

| Method | Description |
| :--- | :--- |
| `thenApply(Function)` | Transform the result |
| `thenAccept(Consumer)` | Consume the result |
| `thenRun(Runnable)` | Run after completion |
| `thenCompose(Function)` | FlatMap — chain dependent futures |
| `thenCombine(other, BiFunction)` | Combine two independent futures |
| `handle(BiFunction)` | Handle result or exception |
| `exceptionally(Function)` | Recover from exception |
| `whenComplete(BiConsumer)` | Observe result or exception without changing it |

### Coordination

| Method | Description |
| :--- | :--- |
| `allOf(cf1, cf2, ...)` | Wait for all to complete |
| `anyOf(cf1, cf2, ...)` | Wait for any to complete |

```java
CompletableFuture.supplyAsync(() -> fetchUser(id))
    .thenApply(User::name)
    .thenAccept(name -> System.out.println("Got: " + name))
    .exceptionally(ex -> { System.err.println(ex); return null; });

// Combine two independent results
CompletableFuture<String> greeting =
    CompletableFuture.supplyAsync(() -> fetchUser(id))
        .thenCombine(
            CompletableFuture.supplyAsync(() -> fetchGreeting(locale)),
            (user, greet) -> greet + " " + user.name()
        );
```

---

## Records

Immutable data carriers with auto-generated `equals()`, `hashCode()`, `toString()`, and accessor methods (since Java 16).

```java
record Point(int x, int y) {}

Point p = new Point(3, 4);
p.x();                    // 3
p.y();                    // 4
System.out.println(p);    // Point[x=3, y=4]
```

### Compact Constructor (validation)

```java
record Range(int start, int end) {
    Range {
        if (start > end) throw new IllegalArgumentException("start > end");
    }
}
```

### Custom Methods

```java
record Person(String name, int age) {
    boolean isAdult() {
        return age >= 18;
    }
}
```

### Records with Generics

```java
record Pair<A, B>(A first, B second) {}

Pair<String, Integer> entry = new Pair<>("key", 42);
```

> [!NOTE]
> Records are implicitly `final`, extend `java.lang.Record`, and cannot extend other classes. They can implement interfaces.

---

## Sealed Classes

Restrict which classes can extend or implement a type (since Java 17).

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}

record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}
final class Triangle implements Shape {
    // ...
}
```

Permitted subclasses must be `final`, `sealed`, or `non-sealed`.

```java
sealed class Account permits SavingsAccount, CheckingAccount {}
final class SavingsAccount extends Account {}
non-sealed class CheckingAccount extends Account {}
```

---

## Pattern Matching

### instanceof Pattern (since Java 16)

Eliminates explicit casting after type checks.

```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// After
if (obj instanceof String s) {
    System.out.println(s.length());
}

// With conditions
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s);
}
```

### Pattern Matching for switch (since Java 21)

```java
static String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0  -> "positive int: " + i;
        case Integer i             -> "non-positive int: " + i;
        case String s              -> "string of length " + s.length();
        case null                  -> "null";
        default                    -> "something else";
    };
}
```

### Record Patterns (since Java 21)

Deconstruct records directly in pattern matching.

```java
record Point(int x, int y) {}

static String format(Object obj) {
    return switch (obj) {
        case Point(int x, int y) when x == 0 && y == 0 -> "origin";
        case Point(int x, int y)                        -> "(%d, %d)".formatted(x, y);
        default                                         -> "not a point";
    };
}

// Nested deconstruction
record Line(Point start, Point end) {}

if (obj instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
    double length = Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
}
```

### Exhaustive switch with Sealed Types

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

double area(Shape shape) {
    return switch (shape) {
        case Circle(var r)        -> Math.PI * r * r;
        case Rectangle(var w, var h) -> w * h;
    };
}
```

---

## Switch Expressions

Return a value from a `switch` (since Java 14).

### Arrow Syntax

```java
String label = switch (day) {
    case MONDAY, FRIDAY    -> "Busy";
    case TUESDAY           -> "Meeting";
    case SATURDAY, SUNDAY  -> "Weekend";
    default                -> "Normal";
};
```

### yield for Multi-Statement Branches

```java
int result = switch (status) {
    case "OK" -> 200;
    case "NOT_FOUND" -> 404;
    case "ERROR" -> {
        log("Server error");
        yield 500;
    }
    default -> throw new IllegalArgumentException("Unknown: " + status);
};
```

---

## Text Blocks

Multi-line string literals with managed indentation (since Java 15).

```java
String json = """
        {
            "name": "Alice",
            "age": 30
        }
        """;

String html = """
        <html>
            <body>
                <p>Hello %s</p>
            </body>
        </html>
        """.formatted(name);

String query = """
        SELECT id, name
        FROM users
        WHERE active = true
        ORDER BY name
        """;
```

> [!TIP]
> The closing `"""` position determines the base indentation. Move it left to include more whitespace, right to strip it.

---

## var (Local Variable Type Inference)

Infers the type from the right-hand side (since Java 10, `var` in lambdas since Java 11).

```java
var list = new ArrayList<String>();         // ArrayList<String>
var map = Map.of("a", 1, "b", 2);          // Map<String, Integer>
var stream = list.stream();                 // Stream<String>

// In for-each
for (var entry : map.entrySet()) {
    System.out.println(entry.getKey());
}

// In lambdas (since Java 11) — enables annotations
list.sort((@NonNull var a, @NonNull var b) -> a.compareTo(b));
```

> [!WARNING]
> `var` cannot be used for method parameters, return types, or fields. It is only for local variables with an initialiser.

---

## New I/O & HTTP Methods

### Files (since Java 11)

| Method | Returns | Description |
| :--- | :--- | :--- |
| `Files.readString(Path)` | `String` | Read entire file as a string |
| `Files.writeString(Path, String)` | `Path` | Write a string to a file |
| `Files.readString(Path, Charset)` | `String` | Read with specific charset |
| `Files.writeString(Path, String, OpenOption...)` | `Path` | Write with options (append, etc.) |

```java
String content = Files.readString(Path.of("config.txt"));
Files.writeString(Path.of("output.txt"), "Hello World");
Files.writeString(Path.of("log.txt"), "entry\n", StandardOpenOption.APPEND);
```

### Path.of (since Java 11)

```java
Path path = Path.of("home", "user", "file.txt");  // home/user/file.txt
Path abs = Path.of("/etc/config.yml");
```

### HTTP Client (since Java 11)

Replaces the legacy `HttpURLConnection`.

```java
HttpClient client = HttpClient.newHttpClient();

// Synchronous GET
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .header("Accept", "application/json")
    .GET()
    .build();

HttpResponse<String> response =
    client.send(request, HttpResponse.BodyHandlers.ofString());
int status = response.statusCode();
String body = response.body();

// Asynchronous GET
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);

// POST with JSON body
HttpRequest post = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("""
        {"name": "Alice", "age": 30}
        """))
    .build();

// Custom client with timeout
HttpClient custom = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .followRedirects(HttpClient.Redirect.NORMAL)
    .build();
```

---

## Helpful NullPointerExceptions

Since Java 14, NPE messages include the exact variable or method call that was `null`.

```text
// Before Java 14
Exception in thread "main" java.lang.NullPointerException

// Since Java 14
Exception in thread "main" java.lang.NullPointerException:
  Cannot invoke "String.length()" because "user.getName()" is null
```

No code changes needed — this is a JVM improvement.

---

## Version Reference

Quick lookup for when each feature became available.

| Feature | Preview | Stable |
| :--- | :--- | :--- |
| `var` (local variables) | — | Java 10 |
| `var` in lambda parameters | — | Java 11 |
| String: `isBlank`, `strip`, `lines`, `repeat` | — | Java 11 |
| `Optional.isEmpty()` | — | Java 11 |
| HTTP Client API | — | Java 11 |
| `Files.readString` / `writeString` | — | Java 11 |
| `Predicate.not()` | — | Java 11 |
| Switch expressions | Java 12 | Java 14 |
| `String.indent()`, `transform()` | — | Java 12 |
| `Collectors.teeing()` | — | Java 12 |
| Text blocks | Java 13 | Java 15 |
| Helpful NullPointerExceptions | — | Java 14 |
| Records | Java 14 | Java 16 |
| `instanceof` pattern matching | Java 14 | Java 16 |
| `Stream.toList()` | — | Java 16 |
| `Stream.mapMulti()` | — | Java 16 |
| Sealed classes | Java 15 | Java 17 |
| Pattern matching for switch | Java 17 | Java 21 |
| Record patterns | Java 19 | Java 21 |
