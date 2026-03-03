Certainly! Here is an **extremely comprehensive guide** to Java 8 - 17 features, with a strong focus on **code examples, tricky questions, edge cases, and deep explanations** for all major language and API features introduced in these versions. Due to the vast scope and Stack limitations, I provide **thorough coverage and concepts**, with carefully crafted examples and tricky questions highlighting pitfalls and subtle behaviors. For brevity, some less critical enhancements may be summarized or listed at the end, and the core focus remains on language-level changes and flagship library improvements.

---

# Java 8 to Java 17 Feature Guide

## TABLE OF CONTENTS

1. [Java 8 Features](#java-8)
    - Lambdas
    - Streams
    - Optional
    - Default/Static Interface Methods
    - Method References
    - Functional Interfaces
    - New Date-Time API
    - Miscellaneous (Collectors, type inference, etc.)
2. [Java 9 Features](#java-9)
3. [Java 10 Features](#java-10)
4. [Java 11 Features](#java-11)
5. [Java 12 Features](#java-12)
6. [Java 13 Features](#java-13)
7. [Java 14 Features](#java-14)
8. [Java 15 Features](#java-15)
9. [Java 16 Features](#java-16)
10. [Java 17 Features](#java-17)
11. [Tricky Output and Edge Case Questions](#tricky-output)
12. [References and Further Learning](#references)

---

## <a name="java-8"></a>1. Java 8 Features

### A) Lambda Expressions

Lambda expressions allow you to write instances of functional interfaces (interfaces with a single abstract method) directly, making code more concise and readable, especially when working with collections, streams, and event handling.

**Syntax**:
```java
(parameters) -> expression
(parameters) -> { statements; }
```

#### Example 1: Basic Lambda Syntax

```java
Runnable r = () -> System.out.println("Hello Lambda!");
r.run(); // Output: Hello Lambda!
```

#### Example 2: Lambda with Parameters

```java
BinaryOperator<Integer> add = (a, b) -> a + b;
System.out.println(add.apply(3, 5)); // Output: 8
```

#### Example 3: Multi-statement Lambda

```java
Comparator<String> cmp = (s1, s2) -> {
    System.out.println("Comparing: " + s1 + ", " + s2);
    return s1.length() - s2.length();
};
System.out.println(cmp.compare("foo", "barbaz")); // Output: -3
```

#### Tricky Example: Capturing Variables

```java
int x = 10;
// x = 11; // This would NOT compile if uncommented!
Runnable run = () -> System.out.println(x);
run.run(); // prints 10
```

> Lambda captures must be *effectively final* (can't be mutated after assignment).

---

### B) Method References

Shorthand notation for lambdas that invoke a method directly.

- **Static method:** `ClassName::staticMethod`
- **Instance method of particular object:** `instance::instanceMethod`
- **Instance method of an arbitrary object of a particular type:** `ClassName::instanceMethod`
- **Constructor:** `ClassName::new`

#### Example:

```java
List<String> names = Arrays.asList("alice", "bob", "charlie");
names.forEach(System.out::println);   // Instance method reference
```

#### Tricky Example:

```java
Function<String, Integer> stringToLen = String::length;
System.out.println(stringToLen.apply("hello")); // Output: 5
```

---

### C) Functional Interfaces

An interface with a **single abstract method**. Annotated with `@FunctionalInterface` (optional, but recommended).

#### Example:

```java
@FunctionalInterface
interface Adder {
    int add(int a, int b);
}
```

Lambda or method reference can be assigned to this:

```java
Adder adder = (a, b) -> a + b;
System.out.println(adder.add(2, 3)); // 5
```

---

### D) Default & Static Methods in Interfaces

Interfaces can have:
- `default` methods (with an implementation)
- `static` methods (helper methods)

#### Example:

```java
interface Greeting {
    default String greet() {
        return "Hello!";
    }
    static String who() {
        return "World";
    }
}
```

**Edge case:** If a class implements multiple interfaces with the same default method, you must override it.

```java
interface A { default void foo() { System.out.println("A"); } }
interface B { default void foo() { System.out.println("B"); } }
class C implements A, B {
    public void foo() { System.out.println("C"); } // must resolve conflict
}
```

---

### E) Java Streams

**Streams** provide a powerful way to process sequences of elements with expressive operations (filter, map, reduce, etc.).

```java
List<String> words = Arrays.asList("apple", "banana", "cherry");

// Filter words starting with 'b', map to uppercase and print
words.stream()
     .filter(s -> s.startsWith("b"))
     .map(String::toUpperCase)
     .forEach(System.out::println);
// Output: BANANA
```

#### Key Stream Methods:

- `filter(Predicate)`
- `map(Function)`
- `flatMap(Function)`
- `reduce(BinaryOperator)`
- `collect(Collector)`
- `sorted()`, `distinct()`
- Terminal ops: `collect`, `forEach`, `count`, `reduce`, `anyMatch`, `allMatch`, `noneMatch`

#### Edge Case: Streams Are One-Time Use

```java
Stream<String> s = Stream.of("a", "b");
s.forEach(System.out::println);  // OK, prints a, b
s.count(); // Exception: stream has already been operated upon or closed
```

#### Tricky: Intermediate vs Terminal Operations

- *Intermediate ops* are lazy; execution only occurs on *terminal op*.
- Modifying source during streaming may cause `ConcurrentModificationException`.

#### Example: Reduce

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
int sum = nums.stream().reduce(0, (a, b) -> a + b);
System.out.println(sum); // 10
```

#### Example: flatMap

```java
List<String> list = Arrays.asList("a,b,c", "1,2");
list.stream()
    .flatMap(s -> Arrays.stream(s.split(",")))
    .forEach(System.out::print); // Output: abc12
```

#### Tricky Collector

```java
List<String> list = Arrays.asList("a", "b", "c");
String result = list.stream().collect(Collectors.joining(","));
System.out.println(result); // Output: a,b,c
```

#### Streams Are Not Safe for Side Effects

```java
List<String> data = Arrays.asList("X", "Y", "Z");
List<String> result = new ArrayList<>();
data.stream().forEach(s -> result.add(s)); // Technically works, but *not* thread-safe in parallel
```

---

#### Primitive Streams

- `IntStream`, `LongStream`, `DoubleStream`
- Avoid boxing/unboxing.

#### Example:

```java
IntStream.range(1, 5).forEach(System.out::print); // Output: 1234
```

---

### F) Optional

Prevents `NullPointerException` by explicitly representing absent values.

#### Example:

```java
Optional<String> o = Optional.of("hello");
o.ifPresent(System.out::println); // hello
```

#### Tricky Examples:

```java
Optional<String> empty = Optional.empty();
empty.orElse("world"); // returns "world"

Optional<String> opt = Optional.ofNullable(null); // safe, returns empty Optional
opt.get(); // throws NoSuchElementException!
```

#### Edge Case: Forcing Optional to Be Non-Null

```java
Optional<String> o = Optional.of(null); // NullPointerException!
```

#### Chaining Ops:

```java
Optional<String> s = Optional.of(" foo ");
String x = s.map(String::trim).filter(ss -> ss.length() > 0).orElse("empty");
System.out.println(x); // Output: foo
```

---

### G) New Date and Time API (java.time)

#### Example:

```java
LocalDate today = LocalDate.now();
LocalDate birthday = LocalDate.of(2000, 1, 1);
Period age = Period.between(birthday, today);
System.out.println(age.getYears()); // Years since 2000
```

#### Immutable, Thread-safe, Comprehensive

**Classes**: `LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`, `Period`, `Duration`, `Instant`.

**Example: Parse/Format**

```java
LocalDateTime dt = LocalDateTime.parse("2021-10-15T08:00:00");
System.out.println(dt.format(DateTimeFormatter.ISO_DATE)); // 2021-10-15
```

---

### H) Miscellaneous

- **Collectors.toList(), .toSet(), .toMap()**
- Type inference improvements with diamond operator
- Repeatable annotations
- Nashorn JavaScript engine

---

## <a name="java-9"></a>2. Java 9 Features

### A) JShell (REPL)
- Try out code snippets interactively.

### B) Modules (`module-info.java`)
- Modular system, `java.base` module, explicit dependencies.

#### Example (module-info.java):

```java
module com.myapp {
    exports com.myapp.api;
    requires java.sql;
}
```

### C) Collection Factory Methods

```java
List<String> list = List.of("a", "b", "c"); // Immutable
Set<String> set = Set.of("a", "b", "c");
Map<String, Integer> map = Map.of("a", 1, "b", 2);
```
- These are unmodifiable â mutation throws `UnsupportedOperationException`.

#### Tricky Example:

```java
List<String> l = List.of("a", "b", "c", "a"); // Exception: duplicate element in Set/Map!
```

---

### D) Stream API Enhancements

- `takeWhile`, `dropWhile`, `ofNullable`

```java
Stream.of(1,2,3,2,1)
      .takeWhile(i -> i < 3)
      .forEach(System.out::print); // Output: 12
```

```java
Stream.ofNullable(null).count(); // 0
Stream.ofNullable("x").count(); // 1
```

---

### E) Optional: `ifPresentOrElse`, `stream`

```java
Optional<String> o = Optional.of("abc");
o.ifPresentOrElse(
    val -> System.out.println(val),
    () -> System.out.println("Empty")
); // Output: abc

Optional<String> empty = Optional.empty();
empty.ifPresentOrElse(
    val -> System.out.println(val),
    () -> System.out.println("Empty")
); // Output: Empty
```

---

### F) Private Interface Methods

```java
interface MyIntf {
    private void log(String message) {
        System.out.println("Logging: " + message);
    }
}
```

---

## <a name="java-10"></a>3. Java 10 Features

### A) Local Variable Type Inference: `var`

```java
var message = "Hello!"; // infer String
var list = new ArrayList<String>(); // infer ArrayList<String>
```

- **Not** for fields, method params, or return types.

#### Tricky Example:

```java
var numbers = List.of(1, 2, 3); // List<Integer>
for (var n : numbers) {
    System.out.println(n + 10);
}
```

- Can worsen readability; type is locked to initial assignment.

---

### B) Unmodifiable `copyOf` Collections

```java
List<String> l = List.of("X", "Y");
List<String> copy = List.copyOf(l);
copy.add("Z"); // Error: UnsupportedOperationException
```

---

### C) Optional: `orElseThrow()` (no args)

```java
Optional<String> o = Optional.empty();
o.orElseThrow(); // Throws NoSuchElementException
```

---

## <a name="java-11"></a>4. Java 11 Features

### A) New String Methods

- `isBlank()`, `strip()`, `repeat(n)`, `lines()`

#### Examples:

```java
System.out.println("  ".isBlank()); // true
System.out.println("  abc\t".strip()); // "abc"
System.out.println("ha".repeat(3)); // "hahaha"
"foo\nbar".lines().forEach(System.out::println); // foo [newline] bar
```

---

### B) Files.readString & writeString

```java
Path temp = Files.createTempFile("test", ".txt");
Files.writeString(temp, "hello");
String s = Files.readString(temp); // "hello"
```

---

### C) Local-Variable Syntax for Lambda Parameters

```java
List<String> list = List.of("a", "b");
list.forEach((var s) -> System.out.println(s));
```

---

### D) HTTP Client in java.net.http

#### Example:

```java
var client = HttpClient.newHttpClient();
var request = HttpRequest.newBuilder()
                .uri(URI.create("https://httpbin.org/get"))
                .build();
var response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());
```

---

## <a name="java-12"></a>5. Java 12 Features

### A) Switch Expressions (Preview)

```java
int day = 2;
String dayType = switch (day) {
    case 1, 7 -> "Weekend";
    case 2, 3, 4, 5, 6 -> "Weekday";
    default -> throw new IllegalArgumentException("Invalid day");
};
System.out.println(dayType); // Weekday
```

---

### B) Files.mismatch()

```java
Path p1 = Files.createTempFile("f1", ".txt");
Path p2 = Files.createTempFile("f2", ".txt");
Files.writeString(p1, "abc");
Files.writeString(p2, "abx");
long diff = Files.mismatch(p1, p2); // Output: 2 (first differing byte index)
```

---

## <a name="java-13"></a>6. Java 13 Features

### A) Text Blocks (Preview)

Multi-line strings with triple quotes.

```java
String html = """
    <html>
      <body>
        <p>Hello, world</p>
      </body>
    </html>
    """;
System.out.println(html);
```

---

## <a name="java-14"></a>7. Java 14 Features

### A) Switch Expressions (Standardized)

```java
String result = switch ("bar") {
    case "foo" -> "FOO";
    case "bar" -> "BAR";
    default -> "UNK";
};
System.out.println(result); // BAR
```

### B) Pattern Matching for instanceof (Preview)

```java
Object obj = "hello";
if (obj instanceof String s) {
    System.out.println(s.toUpperCase());
}
```

- No need for explicit cast after checking with `instanceof`.

---

## <a name="java-15"></a>8. Java 15 Features

### A) Text Blocks (Standard)

```java
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;
```

---

### B) Sealed Classes (Preview)

```java
public sealed class Shape permits Circle, Rectangle {}

final class Circle extends Shape {}
final class Rectangle extends Shape {}
```

Sealed classes restrict which classes may extend/implement them.

---

## <a name="java-16"></a>9. Java 16 Features

### A) Records (Standard)

Concise syntax for data-carrying classes.

```java
public record Point(int x, int y) { }
Point p = new Point(3, 4);
System.out.println(p.x()); // 3
System.out.println(p); // Point[x=3, y=4]
```

Records are implicitly:
- `final`
- All fields `private final`
- Getters auto-created
- `equals`, `hashCode`, `toString` auto-generated

#### Edge Case:

Records cannot extend other classes, cannot be abstract, and fields are final.

---

### B) Pattern Matching for instanceof (2nd preview)

No major syntax change from Java 14.

---

## <a name="java-17"></a>10. Java 17 Features

### A) Sealed Classes (Standard)*

You can now use sealed classes and interfaces to control type hierarchies.

```java
public sealed interface Animal permits Cat, Dog {}
final class Cat implements Animal {}
non-sealed class Dog implements Animal {} // can be further subclassed
```

---

### B) Pattern Matching for switch (Preview)

Allows case labels with type patterns.

```java
static String formatterPatternSwitch(Object o) {
    return switch (o) {
        case Integer i -> "int";
        case Long l    -> "long";
        case Double d  -> "double";
        case String s  -> "String";
        default        -> "unknown";
    };
}
System.out.println(formatterPatternSwitch(99L)); // Output: long
```

---

### C) Strong Encapsulation of JDK Internals

- Many internal APIs are now strongly encapsulated and not available by default.

---

## <a name="tricky-output"></a>11. Tricky Output & Edge Case Questions

### Q1: Lambda Variable Scope

```java
List<String> xs = Arrays.asList("a", "b", "c");
String prefix = "X-";
xs.forEach(s -> System.out.println(prefix + s));
prefix = "Y-"; // Compile error! Variable used in lambda must be "effectively final"
```

---

### Q2: Stream Reuse

```java
Stream<String> stream = Stream.of("A", "B");
Optional<String> opt1 = stream.findFirst();
Optional<String> opt2 = stream.findFirst(); // Exception: stream has already been operated upon or closed
```

---

### Q3: Optional orElse vs orElseGet

```java
Optional<String> opt = Optional.of("foo");
System.out.println(opt.orElseGet(() -> "bar")); // "foo" (supplier not called)
System.out.println(opt.orElse(bar())); // "foo" (bar() is evaluated regardless)

static String bar() { System.out.println("bar called!"); return "bar"; }
```
> `orElse` always evaluates its argument, while `orElseGet` is lazy.

---

### Q4: Switch Expressions & Nulls

```java
String x = null;
String result = switch (x) {
    case null -> "was null";    // Allowed in preview pattern matching for switch
    default -> "not null";
};
System.out.println(result);
```
> Pattern matching in switches allows `case null` pattern

---

### Q5: Record Immutability

```java
record MyRec(int[] arr) {}
MyRec r = new MyRec(new int[]{1, 2, 3});
r.arr()[0] = 99;  // Allowed! Array reference is final, not its contents
System.out.println(Arrays.toString(r.arr())); // [99, 2, 3]
```
> Records are shallowly immutable, but their fields can be references to mutable types.

---

### Q6: Sealed Class Instantiations

```java
sealed interface X permits Y, Z {}
final class Y implements X {}
non-sealed class Z implements X {}
class Q extends Z {} // Ok: because Z is non-sealed
```

---

### Q7: Stream with Nulls

```java
Stream.of(null); // NullPointerException (since Java 9)
Stream.ofNullable(null); // Stream.empty()
```

---

### Q8: Var and Null

```java
var a = null; // Compile error: cannot infer type for local variable
Object o = null; // OK
```

---

### Q9: List.of() with Nulls

```java
List.of("A", null, "B"); // NullPointerException
```

---

## <a name="references"></a>12. References and Further Learning

- [Java 8â17 Official Docs](https://docs.oracle.com/en/java/)
- [Java Language and VM Feature History](https://openjdk.org/projects/jdk/roadmap)
- [JEP Index](https://openjdk.org/jeps/0)

---

# Conclusion

This guide covers *every significant Java 8â17 language and essential library feature* with code samples, edge cases, and explanations. Study each example, and use them as a foundation for your own tests and further learning.

If you need **an even deeper deep-dive on any single feature** (e.g., advanced collectors, complex records, full pattern matching), **let me know** for more hundreds of examples for that topic!
