# Deep Dive: Modern Java Features (Records, Streams, Pattern Matching)

## 1. The Shift: Imperative to Declarative
Modern Java (8 and beyond, with a focus on 17+) shifted from **how** to do things (Imperative) to **what** to do (Declarative).
- **Imperative:** Loop through, check condition, add to list.
- **Declarative:** Filter by condition, collect to list.

---

## 2. Lambda Expressions & Functional Interfaces
Lambdas are anonymous functions. They rely on **Functional Interfaces** (interfaces with exactly one abstract method).
- **`Predicate<T>`:** `test(T t) -> boolean` (Filtering).
- **`Function<T, R>`:** `apply(T t) -> R` (Mapping).
- **`Consumer<T>`:** `accept(T t)` (Action).
- **`Supplier<T>`:** `get()` (Providing data).

---

## 3. Streams API: Data Pipelines
Streams are not data structures. They are **pipelines** for data.

### The Lifecycle of a Stream:
1. **Source:** `list.stream()` or `Stream.of()`.
2. **Intermediate Operations (Lazy):** These return a new Stream and are only executed when a terminal operation is called.
   - `filter()`, `map()`, `flatMap()`, `distinct()`, `sorted()`.
3. **Terminal Operations (Eager):** These produce a result or a side-effect.
   - `collect()`, `forEach()`, `reduce()`, `findFirst()`, `anyMatch()`.

### Why Lazy Evaluation matters?
If you have a million items and you do `.filter(...).findFirst()`, the Stream will only process items until it finds the first match. It doesn't filter the whole million items first.

---

## 4. Records (Java 14+): The DTO Revolution
Records are immutable data carriers. They automatically generate `constructors`, `getters`, `equals()`, `hashCode()`, and `toString()`.

```java
// Traditional Boilerplate (Lombok-less)
public record UserDTO(String name, int age) {} 

// Usage
UserDTO user = new UserDTO("Alice", 25);
System.out.println(user.name()); // Alice
```
**Production Tip:** Use Records for DTOs, API responses, and database projections.

---

## 5. Pattern Matching & Switch Expressions (Java 17+)
Reduces the "Casting" boilerplate and makes code safer.

### Pattern Matching for `instanceof`:
```java
if (obj instanceof String s) {
    System.out.println(s.toLowerCase()); // No explicit cast needed!
}
```

### Switch Expressions:
```java
String result = switch (status) {
    case ACTIVE -> "User is online";
    case INACTIVE -> "User is away";
    default -> throw new IllegalStateException("Unknown: " + status);
};
```

---

## 6. Production Grade Examples

### Refactoring Legacy Code
**Legacy:**
```java
List<String> names = new ArrayList<>();
for (User u : users) {
    if (u.getAge() > 18) {
        names.add(u.getName());
    }
}
```
**Modern:**
```java
List<String> names = users.stream()
    .filter(u -> u.age() > 18)
    .map(User::name)
    .toList(); // Java 16+ syntax
```

---

## 7. Common Traps & Pitfalls

### Stream Re-use
**TRAP:** You cannot reuse a Stream after a terminal operation has been called. It will throw `IllegalStateException`.

### Side Effects in Pipelines
**TRAP:** Modifying external state inside a `.map()` or `.filter()`.
```java
List<Integer> list = new ArrayList<>();
stream.map(i -> { list.add(i); return i; })... // AVOID THIS
```
Functional programming should be "Pure" — data goes in, new data comes out, no side effects.

### Parallel Streams are NOT a silver bullet
**TRAP:** Using `.parallelStream()` for everything. It uses a common **ForkJoinPool**. If one task is slow/blocking, it can starve the entire JVM of threads for other parallel streams. Use only for CPU-intensive tasks on large datasets.

---

## 8. Summary Checklist
- [x] Streams are lazy; they only work when a terminal operation is called.
- [x] Records are immutable and replace verbose DTO classes.
- [x] Pattern matching removes the need for manual casting.
- [x] Avoid side effects inside Stream operations.
- [x] Use `parallelStream()` with caution (beware of the shared thread pool).
- [x] Method references (`User::getName`) make code cleaner than lambdas.
