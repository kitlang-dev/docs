# Kit Style Guide

This style guide provides in-depth styling conventions for Kit code, based on analysis of:
- the Kit standard library
- actual Kit projects

This guide aims to describe a common Kit style. Following these conventions ensures code readability, maintainability, and consistency across the Kit ecosystem.

## Naming Conventions

| What | Convention | Example |
| --- | --: | --- |
| Types (structs, enums, traits, aliases) |                     **PascalCase** | `struct PositionComponent { }`<br>`enum Option[T] { }` |
| Functions & methods                     |                      **camelCase** | `function getComponent() { }`                          |
| Local variables & fields                |                      **camelCase** | `var entityName: CString`                              |
| Static constants (`static const`)       |                      **camelCase** | `static const internalArrayThreshold: Float = 0.7`     |
| Static mutable (`static var`)           | **camelCase** (for counters/state) | `static var id: Int = 0`                               |
| Files                                   |      **lowercase-with-dashes.kit** | `my-component.kit`                                     |
| Modules (imports)                       |                      **lowercase** | `import kit.map;`                                      |
| Generic type params                     |        **Single uppercase letter** | `struct Map[K: Hashable, V] { }`                       |

## Naming rules (concise)

* **Types:** `PascalCase` — start with uppercase, no separators.
  Example: `PositionComponent`, `Result[T, E]`.

* **Functions & methods:** `camelCase` — lower-first, then capital letters per word.
  Example: `getComponent()`, `addComponent()`.

* **Variables & fields:** `camelCase` for locals/fields. Prefer descriptive names but keep them short.
  Example: `componentCount`, `entityName`.

* **Static members:**

  * `static const`: `camelCase` (treat like constants scoped to the file/type).
    Example: `static const maxLights: Int = 4`.
  * `static var`: `camelCase` (used for runtime counters/state).
    Example: `static var lightsCount: Int = 0`.

* **Files & modules:**

  * File names: `lowercase-with-dashes.kit` (easy to read in terminals).
  * Module names: all lowercase (matches import style).

* **Type parameters:** single uppercase letters — `T`, `K`, `V`, `E`.
  Example: `struct List[T] { }`

## File layout

1. **Imports** (first)

   ```kit
   import kit.hash;
   import kit.map;
   import kit.option;
   ```

2. **C includes** (if needed, immediately after imports)

   ```c
   include "stdlib.h";
   include "raylib.h";
   ```

3. **Types** (group related types; order by dependency)

   * enums → traits → structs → abstract types / aliases

4. **Implementations** (trait implementations, `implement` blocks)

5. **Top-level functions** (helpers and `main` at the bottom)

Example:

```kit
import kit.hash;
include "raylib.h";

enum BlendMode { }
trait Component { }
struct PositionComponent { }

implement Component for PositionComponent { }

function main() { }
```

## Visibility & API surface

* Prefer **explicit** visibility:

  * `public` for things you want in the module API.
  * `private` for internal helpers.
* Default visibility varies — don’t rely on it. Be explicit.

Examples:

```kit
public function getComponent() { }
private function internalHelper() { }
```

## Type definition example

```kit
struct PositionComponent {
    public var x: Float;
    public var y: Float;
    private var allocator: Box[Allocator];

    public static function new(allocator: Box[Allocator]): PositionComponent {
        return struct Self {
            x: 0.0,
            y: 0.0,
            allocator,
        };
    }
}
```

**Guidelines:**
- List fields before methods
- Group public fields together, then private fields
- Place static methods before instance methods
- Use `Self` to reference the type within its own definition

### Enums

```kit
enum Option[T] {
    Some(value: T);
    None;

    public function isSome(): Bool {
        match this {
            None => return false;
            default => return true;
        }
    }

    public function unwrap(): T {
        match this {
            Some(value) => return value;
            default => panic("unwrap: unexpected missing value");
        }
    }
}
```

**Guidelines:**
- List variants with associated values first
- Include methods for common operations (isX, unwrap, etc.)
- Use pattern matching in methods

### Traits

```kit
trait Component {
    function typeIdentifier(): CString;

    public function base(): Ptr[Void] {
        return &this;
    }
}

trait Hashable {
    function hash(): Int;
}

trait Iterable(IteratorT) {
    public function iterator(): Box[Iterator[IteratorT]];
}
```

**Guidelines:**
- Define required methods without implementation
- Provide default implementations when appropriate
- Use type parameters for associated types

## Functions and Methods

### Function Signatures

```kit
// Static function (constructor)
public static function new(allocator: Box[Allocator], capacity: Int): Map[K, V] using implicit allocator {
    var internalArray: Array[KeyValuePair[K, V]] = Array.new(capacity);
    return struct Self {
        allocator,
        internalArray,
    };
}

// Instance method
public function get(key: K): Option[V] {
    var index = this.findLocation(key);
    if this.internalArray[index].isActive {
        return Some(this.internalArray[index].value);
    }
    return None;
}

// Private method
private function findLocation(key: K): Int {
    var hash = key.Hashable.hash() % this.internalArray.length;
    while this.internalArray[hash].isActive && this.internalArray[hash].key != key {
        hash = (hash + 1) % this.internalArray.length;
    }
    return hash;
}
```

**Guidelines:**
- Always specify return types explicitly
- Use descriptive parameter names
- Group static methods before instance methods
- Use `public` for API, `private` for internals
- Use `using implicit allocator` for allocator-aware functions

### Return Values

```kit
// Return Option for potentially missing values
public function get(key: K): Option[V] {
    if this.exists(key) {
        return Some(this.internalArray[index].value);
    }
    return None;
}

// Return Result for operations that can fail
enum Result[T, E] {
    Ok(value: T);
    Error(error: E);
}

// Return the type itself when always successful
public function length(): Int {
    return this.length;
}
```

**Guidelines:**
- Use `Option[T]` for nullable return values
- Use `Result[T, E]` for fallible operations
- Avoid returning raw pointers when `Option` is more appropriate

## Traits and Implementations

### Trait Definitions

```kit
trait Hashable {
    function hash(): Int;
}

trait Allocator {
    function alloc(n: Size): Ptr[Void];
    function free(ptr: Ptr[Void]): Void;

    function calloc(n: Size): Ptr[Void] {
        var ptr = this.alloc(n);
        memset(ptr, 0, n);
        return ptr;
    }
}
```

### Implementing Traits

```kit
implement Hashable for Int {
    function hash(): Int {
        return this;
    }
}

implement Component for PositionComponent {
    function typeIdentifier(): CString {
        return "Position";
    }
}

implement Allocator for SimpleAllocator {
    public function alloc(n: Size): Ptr[Void] {
        return (this.alloc)(n);
    }

    public function free(ptr: Ptr[Void]) {
        (this.free)(ptr);
    }
}
```

**Guidelines:**
- Implement traits in the same file as the type when possible
- Use trait bounds on generic types: `struct Map[K: Hashable, V]`
- Provide trait implementations for built-in types in common.kit or similar

## Match Expressions

Match expressions are a core feature in Kit and should be used extensively:

```kit
enum Option[T] {
    Some(value: T);
    None;
}

public function isSome(): Bool {
    match this {
        None => return false;
        default => return true;
    }
}
```

**Guidelines:**
- Use match for exhaustive pattern matching on enums
- Use wildcard patterns (`_` or `default`) for unhandled cases
- Prefer match over nested if-else chains
- Use match for type-safe unwrapping of Option and Result types

## Pattern Matching and Rules

Kit's rule system allows for compile-time pattern matching and rewriting:

```kit
// Operator overloading
rules {
    ($this + $other) => Vector2Add($this, $other);
    ($this - $other) => Vector2Subtract($this, $other);
}

// Custom operators
rules {
    ($this ?? $other) => $this.or($other);
}

// Property accessors
rules {
    ($this.forward) => $this.getForward();
    ($this.forward = $vector) => $this.setForward($vector);
}

// Index-based access
rules {
    ($this[$k] = $v) => $this.put($k, $v);
    ($this[$k]) => $this.get($k).unwrap();
}

// Iterator optimization
rules {
    (for $ident in $this {$e}) => {
        var __length = $this.length;
        for __i in 0 ... __length {
            var $ident = $this.data[__i];
            {$e}
        }
    }
}
```

**Guidelines:**
- Use rules to define custom operators and syntactic sugar
- Optimize common patterns (iteration, indexing) with rules
- Keep rules simple and focused
- Use rules to provide ergonomic APIs (property access, operators)

## Documentation

### Comments

```kit
/**
 * A List[T] is an immutable singly linked list.
 *
 * This type provides functional-style list operations with O(1) cons
 * and O(n) access by index.
 */

/**
 * Returns a new list with the provided value added to the front.
 * @param {T} value - The value to prepend
 * @return {List[T]} A new list with value as the head
 */
public function cons(value: T): Self {
    return Cons(value, &this);
}

// Inline comments for clarification
var len = 0; // Current length counter
```

**Guidelines:**
- Use block comments (`/** ... */`) for documentation
- Describe what and why, not how
- Document public APIs thoroughly
- Use `@param` and `@return` tags for function documentation
- Keep comments up-to-date with code changes

### File Headers

A files had the following"
```kit
/**
 * {Project Description Here}
 * @author walker84837
 */
```

## Type Safety and Memory Management

### Box and Pointer Types

```kit
// Box for owned, heap-allocated values
struct Entity {
    private var components: Map[CString, Box[Component]];
    private var allocator: Box[Allocator];
}

// Ptr for references and unsafe access
public function base(): Ptr[Void] {
    return &this;
}

public function get(key: K): Option[Ptr[V]] {
    var index = this.findLocation(key);
    if this.internalArray[index].isActive {
        return Some(&this.internalArray[index].value);
    }
    return None;
}
```

**Guidelines:**
- Use `Box[T]` for owned heap allocations
- Use `Ptr[T]` for references and unsafe operations
- Use `Option[Ptr[T]]` when a pointer might be null
- Prefer `Option[T]` over `Ptr[T]` when possible

### Allocators

```kit
public static function new(allocator: Box[Allocator]): Map[K, V] using implicit allocator {
    var internalArray: Array[KeyValuePair[K, V]] = Array.new(capacity);
    return struct Self {
        allocator,
        internalArray,
    };
}

function copy(allocator: Box[Allocator]): Array[T] using implicit allocator {
    var a = Self.new(this.length);
    this.blit(a, 0, this.length);
    return a;
}
```

**Guidelines:**
- Explicitly pass allocators to heap-allocating functions
- Use `using implicit allocator` when appropriate
- Document allocator requirements in function docs
- Ensure proper cleanup (call `.free()` or `.destroy()`)

### Abstract Types

```kit
#[promote] abstract RVector3: Vector3 {
    public static function new(n1: Float, n2: Float, n3: Float): RVector3 {
        return struct RVector3 {
            x: n1,
            y: n2,
            z: n3
        };
    }

    rules {
        ($this + $other) => Vector3Add($this, $other);
    }
}

abstract String: Slice[Char] {
    public static function fromCString(allocator: Box[Allocator], source: CString): String {
        var length = source.length;
        var data: CString = allocator.alloc(length + 1);
        strcpy(data, source);
        return struct String {
            length,
            data,
        };
    }
}
```

**Guidelines:**
- Use abstract types to wrap C types
- Provide ergonomic Kit-style APIs
- Use `#[promote]` to enable automatic promotions
- Implement rules for operator overloading

## Common Patterns

### Option Pattern

Use `Option[T]` to represent nullable or optional values:

```kit
enum Option[T] {
    Some(value: T);
    None;
}

// Check if value exists
if option.isSome() {
    var value = option.unwrap();
    // use value
}

// Provide default
var value = option.or(defaultValue);

// Chain operations
var result = maybeNumber.map(fn { return x * 2; });
```

### Result Pattern

Use `Result[T, E]` for fallible operations:

```kit
enum Result[T, E] {
    Ok(value: T);
    Error(error: E);
}

// Handle errors gracefully
match result {
    Ok(data) => processData(data),
    Error(err) => handleError(err),
}
```

### Builder Pattern

Use fluent method chaining for complex construction:

```kit
var builder = StringBuilder.new();
builder.append("Hello");
builder.append(" ");
builder.append("World");
var result = builder.toString();
```

### Resource Management Pattern

Use `using` with allocators for RAII-style cleanup:

```kit
public static function new(allocator: Box[Allocator]): Map[K, V] using implicit allocator {
    var internalArray: Array[KeyValuePair[K, V]] = Array.new(capacity);
    return struct Self {
        allocator,
        internalArray,
    };
}

public function free() {
    this.internalArray.free();
}
```

### Trait Object Pattern

Use `Box[Trait]` for dynamic dispatch:

```kit
var system: Box[System] = struct MovementSystem {};
system.update(delta, engine);
```

## Anti-Patterns

### Don't: Return raw pointers when Option is appropriate

```kit
// BAD: Can return null
public function get(key: K): Ptr[V] {
    if exists {
        return &value;
    }
    return null;
}

// GOOD: Explicit nullability
public function get(key: K): Option[Ptr[V]] {
    if exists {
        return Some(&value);
    }
    return None;
}
```

### Don't: Use panics for expected conditions

```kit
// BAD: Panics on common case
public function get(key: K): V {
    return this.internalArray[index].unwrap();
}

// GOOD: Returns Option
public function get(key: K): Option[V] {
    if this.internalArray[index].isActive {
        return Some(this.internalArray[index].value);
    }
    return None;
}
```

### Don't: Expose mutable state unnecessarily

```kit
// BAD: Public mutable field
struct Counter {
    public var count: Int;
}

// GOOD: Private field with accessor
struct Counter {
    private var count: Int;

    public function get(): Int {
        return this.count;
    }
}
```

### Don't: Ignore return values

```kit
// BAD: Ignoring result
map.put(key, value);

// GOOD: Handle or explicitly ignore
_ = map.put(key, value);
```

## Abstract Types and C Interop

Kit provides abstract types to wrap C libraries safely:

```kit
// Promote allows implicit conversion
#[promote] abstract RVector3: Vector3 {
    public static function new(n1: Float, n2: Float, n3: Float): RVector3 {
        return struct RVector3 {
            x: n1,
            y: n2,
            z: n3
        };
    }

    public function normalize(): RVector3 {
        this.set(Vector3Normalize(this));
        return this;
    }

    rules {
        ($this + $other) => Vector3Add($this, $other);
        ($this - $other) => Vector3Subtract(this, $other);
    }
}
```

**Guidelines for abstract types:**
- Use `#[promote]` for automatic conversions from C types
- Provide static constructors (e.g., `new()`)
- Add idiomatic Kit methods over C functions
- Define rules for operator overloading
- Prefix with project initial (e.g., `RVector3`, `RColor`)

## Testing

```kit
// Simple test pattern
function testOptionSome() {
    var opt = Some(42);
    assert(opt.isSome());
    assert(opt.unwrap() == 42);
}

function testOptionNone() {
    var opt: Option[Int] = None;
    assert(opt.isNone());
}

function runAllTests() {
    testOptionSome();
    testOptionNone();
    printf("All tests passed!\n");
}

function main() {
    runAllTests();
}
```

## Best Practices

### Error Handling

```kit
// Prefer Option/Result over null/panics
public function get(key: K): Option[V] {
    if this.exists(key) {
        return Some(this.internalArray[index].value);
    }
    return None;
}

// Use unwrap sparingly, and document assumptions
public function unwrap(): T {
    match this {
        Some(value) => return value;
        default => panic("unwrap: unexpected missing value");
    }
}

// Use or() for providing defaults
public function or(other: T): T {
    match this {
        Some(v) => return v;
        default => return other;
    }
}
```

### Type Inference

```kit
// Let the compiler infer types when obvious
var entity = Entity.new("Player");

// But be explicit when it improves clarity
var components: Map[CString, Box[Component]] = Map.new(10);
```

### Function Length and Complexity

- Keep functions focused and short (ideally < 50 lines)
- Extract complex logic into helper functions
- Use meaningful names to reduce need for comments
- Prefer composition over complex nested logic

### Immutable by Default

```kit
// Prefer returning new values
public function add(other: Vector2): Vector2 {
    return Vector2Add(this, other);
}

// Over mutable state when safe and necessary
public function addInPlace(other: Vector2) {
    this.x += other.x;
    this.y += other.y;
}
```

### Trait Usage

```kit
// Use traits to define shared behavior
trait Component {
    function typeIdentifier(): CString;
}

// Use trait bounds to constrain generics
struct Map[K: Hashable, V] {
    // ...
}

// Implement traits for built-in types
implement Hashable for Int {
    function hash(): Int {
        return this;
    }
}
```

### Performance Considerations

- Pre-allocate collections when size is known: `Array.new(100)`
- Use iteration rules to optimize common patterns
- Minimize allocations in hot paths
- Use `Box` for large values, stack for small ones
- Profile before optimizing
- Use `using implicit allocator` to reduce boilerplate
- Prefer stack allocation for small, short-lived structs
- Use `Ptr` arithmetic carefully in performance-critical sections

### Memory Safety

```kit
// Always check Option/Result before unwrapping
match option {
    Some(value) => process(value),
    None => handleMissing(),
}

// Use Ptr only when necessary
var ptr: Ptr[MyStruct] = &myStruct;

// Free explicitly allocated resources
array.free();
stringBuffer.free();
```

### Function Organization

```kit
struct MyStruct {
    // 1. Fields first
    public var id: Int;
    private var data: Array[Int];

    // 2. Static constructors
    public static function new(): MyStruct { }
    public static function withId(id: Int): MyStruct { }

    // 3. Public API methods
    public function get(): Int { }
    public function set(value: Int) { }

    // 4. Private helper methods
    private function validate() { }
    private function cleanup() { }
}
```

### Type Inference Guidelines

```kit
// Let the compiler infer when obvious
var entity = Entity.new("Player");  // Type is clear from new()
var result = Some(42);  // Type is clear from value

// Be explicit when it improves clarity or needed for APIs
var components: Map[CString, Box[Component]] = Map.new(10);

// Explicit types for complex generics
var systems: Map[CString, Box[System]] = Map.new(5);
```

### Generic Type Parameters

```kit
// Single type parameter
struct List[T] { }

// Multiple type parameters
struct Map[K: Hashable, V] { }

// Result type with error type
enum Result[T, E] {
    Ok(value: T);
    Error(error: E);
}

// Trait with associated type
trait Iterable(IteratorT) {
    public function iterator(): Box[Iterator[IteratorT]];
}
```

### Operator Overloading via Rules

```kit
rules {
    // Arithmetic operators
    ($this + $other) => Vector3Add($this, $other);
    ($this - $other) => Vector3Subtract($this, $other);
    ($this * $scalar) => Vector3Multiply(this, $scalar);

    // Compound assignment
    ($this += $other) => $this.add($other);
    ($this -= $other) => $this.subtract($other);

    // Comparison operators
    ($this == $other) => this.equals(other);
    ($this != $other) => !this.equals(other);

    // Indexing
    ($this[$i]) => this.get(i).unwrap();
    ($this[$i] = $v) => this.set(i, $v);

    // Null coalescing
    ($this ?? $other) => $this.or($other);
}
```

## Quick Reference

### Naming Summary

| Category | Convention | Example |
|----------|------------|---------|
| Types | PascalCase | `PositionComponent`, `Option[T]` |
| Functions | camelCase | `getComponent()`, `update()` |
| Variables | camelCase | `entityCount`, `delta` |
| Constants | camelCase | `internalArrayThreshold` |
| Type Params | Single uppercase | `T`, `K`, `V`, `E` |
| Files | kebab-case | `my-component.kit` |
| Modules | lowercase | `kit.map` |

### Common Type Patterns

```kit
// Option for nullable values
enum Option[T] {
    Some(value: T);
    None;
}

// Result for errors
enum Result[T, E] {
    Ok(value: T);
    Error(error: E);
}

// Either for two possible types
enum Either[A, B] {
    Left(a: A);
    Right(b: B);
}

// List for immutable sequences
enum List[T] {
    Cons(head: T, tail: Ptr[List[T]]);
    Empty;
}
```

### Match Expression Templates

```kit
// Exhaustive enum matching
match option {
    Some(value) => handleValue(value),
    None => handleEmpty(),
}

// Guard patterns
match number {
    n if n > 0 => handlePositive(n),
    n if n < 0 => handleNegative(n),
    _ => handleZero(),
}

// Wildcard pattern
match result {
    Ok(data) => processData(data),
    Error(_) => handleError(),
}
```

### Standard Library Functions

```kit
// Option methods
opt.isSome()      // Check if has value
opt.isNone()      // Check if empty
opt.unwrap()      // Get value or panic
opt.or(default)   // Get value or default

// Result methods
result.isOk()     // Check if success
result.isError()  // Check if failure
result.unwrap()   // Get value or panic

// String methods
cstring.length    // Get string length
cstring.startsWith(other)  // Check prefix
cstring.endsWith(other)    // Check suffix

// Array operations
array.length      // Get length
array[i]          // Access element
array[i ... j]    // Slice
```

### Common Rules

```kit
// Property accessors
rules {
    ($this.name) => $this.getName();
    ($this.name = $v) => $this.setName($v);
}

// Method aliases
rules {
    ($this.length) => $this.getLength();
}

// Collection indexing
rules {
    ($this[$k]) => $this.get($k).unwrap();
    ($this[$k] = $v) => $this.put($k, $v);
}
```

- Pre-allocate collections when size is known: `Array.new(100)`
- Use iteration rules to optimize common patterns
- Minimize allocations in hot paths
- Use `Box` for large values, stack for small ones
- Profile before optimizing

### Pattern Matching Best Practices

```kit
// Exhaustive matching on enums
match option {
    Some(value) => process(value),
    None => handleEmpty(),
}

// Use guard patterns when appropriate
match result {
    Ok(value) if value > 0 => handlePositive(value),
    Ok(value) => handleNonPositive(value),
    Error(err) => handleError(err),
}

// Prefer match over nested if-else
match state {
    EntityState.COMPONENT_ADDED => handleAdd(),
    EntityState.COMPONENT_REMOVED => handleRemove(),
    _ => {},
}
```

## Example: Complete Kit File

```kit
/**
 * A simple counter component for the ECS system
 * Demonstrates proper Kit style for components, traits, and systems
 */

import ecs;

/**
 * Counter component that tracks a value with a maximum limit
 */
struct CounterComponent {
    public var count: Int;
    public var max: Int;

    /**
     * Create a new counter with initial and maximum values
     * @param {Int} initial - Starting count value
     * @param {Int} max - Maximum allowed count
     * @return {CounterComponent} New counter instance
     */
    public static function new(initial: Int, max: Int): CounterComponent {
        return struct Self {
            count: initial,
            max,
        };
    }

    /**
     * Increment counter if below maximum
     */
    public function increment() {
        if this.count < this.max {
            this.count++;
        }
    }

    /**
     * Decrement counter if above zero
     */
    public function decrement() {
        if this.count > 0 {
            this.count--;
        }
    }

    /**
     * Reset counter to zero
     */
    public function reset() {
        this.count = 0;
    }

    /**
     * Check if counter is at maximum
     * @return {Bool} True if count equals max
     */
    public function isFull(): Bool {
        return this.count == this.max;
    }

    /**
     * Check if counter is empty
     * @return {Bool} True if count equals zero
     */
    public function isEmpty(): Bool {
        return this.count == 0;
    }
}

implement Component for CounterComponent {
    function typeIdentifier(): CString {
        return "Counter";
    }
}

/**
 * System that updates all counter components each tick
 */
struct CounterSystem {
    public var totalTicks: Int = 0;
    private var debug: Bool = false;
}

implement System for CounterSystem {
    /**
     * Update all counter components
     * @param {Float} delta - Time since last frame
     * @param {Ptr[Engine]} engine - Reference to ECS engine
     */
    public function update(delta: Float, engine: Ptr[Engine]): Void {
        var entities = engine.entitiesForComponent("Counter");

        for entity in entities {
            var counter = entity.getComponent("Counter").unwrap().base() as Ptr[CounterComponent];
            counter.increment();

            if this.debug {
                printf("Entity %s: count = %d/%d\n", entity.name, counter.count, counter.max);
            }
        }

        this.totalTicks++;
    }

    function typeIdentifier(): CString {
        return "Counter";
    }

    /**
     * Enable debug logging
     */
    public function enableDebug() {
        this.debug = true;
    }

    /**
     * Disable debug logging
     */
    public function disableDebug() {
        this.debug = false;
    }
}

/**
 * Example usage
 */
function exampleUsage() {
    // Create engine
    var engine = Engine.new();

    // Create entities
    var player = Entity.new("Player");
    var enemy = Entity.new("Enemy");

    // Create components
    var playerCounter = CounterComponent.new(0, 100);
    var enemyCounter = CounterComponent.new(50, 200);

    // Add components to entities
    player.addComponent(playerCounter);
    enemy.addComponent(enemyCounter);

    // Add entities to engine
    engine.addEntity(player);
    engine.addEntity(enemy);

    // Create system
    var counterSystem = struct CounterSystem {};
    counterSystem.enableDebug();

    // Register system
    engine.addSystem("Counter", counterSystem);

    // Simulate game loop
    for i in 0 ... 10 {
        engine.update(0.016);  // ~60 FPS
    }
}
```

## Code Formatting

### Indentation and Spacing

```kit
// Use 4 spaces for indentation
struct MyStruct {
    public var field: Int;

    public function method() {
        if condition {
            // indented with 4 spaces
            doSomething();
        }
    }
}
```

### Line Length

- Aim for 80-100 characters per line
- Break long lines at logical points
- Prefer readability over strict limits

```kit
// Break long function calls
var result = veryLongFunctionName(
    veryLongArgumentName,
    anotherLongArgument
);

// Break long match expressions
match veryLongEnumVariable {
    VeryLongVariantName(v1, v2) => {
        handleVariant(v1, v2);
    }
    _ => {}
}
```

### Blank Lines

- One blank line between functions
- One blank line between type definitions
- Two blank lines between top-level sections

```kit
struct FirstType { }

function firstFunction() { }

struct SecondType { }

function secondFunction() { }
```

### Braces

```kit
// Single-line if: no braces
if condition doSomething();

// Multi-line if: use braces
if condition {
    doSomething();
    doSomethingElse();
}

// Always use braces for match
match option {
    Some(v) => {},
    None => {},
}
```

### Comments

```kit
// Use inline comments sparingly
var x = 5;  // Counter

// Prefer explanatory comments above code
// Initialize counter to zero
var x = 0;

// Block comments for complex logic
/**
 * This function performs a complex calculation
 * involving multiple steps. First it validates input,
 * then applies the algorithm, and finally formats output.
 */
public function complexCalculation(input: Input): Output {
    // implementation
}
```

Following this style guide will ensure your Kit code is readable, maintainable, and consistent with the broader Kit ecosystem.
